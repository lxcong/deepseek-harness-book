# 第 12 章 取消与错误恢复

取消是 agent 系统里最诚实的试金石:LLM 流断在半途、工具跑到一半、用户按下停止、宿主整体卸载——每条路径都必须收敛到一个日志闭合、资源归零的状态。DeepSeek Harness 把这件事拆成三层:`Agent.cancel()` 的**类型化取消因由**与信号传播,`turn()` 的 **catch/finally 收敛**保证 `turn/end` 必落盘,`index.ts` 的**三方熔断与记忆化拆除**保证生命周期无论从哪个方向撕开都能回卷。本章沿这三层读 `agent-loop` 的中断与异常路径,最后看 `invariant.ts` 如何让「恢复对了」变成可被机器检查的命题。

## 问题背景

朴素实现的取消通常是一个 `this.cancelled = true` 加满地 `if (this.cancelled) return`。它会在这些地方漏:

1. **取消原因不可区分**。用户主动停、父 agent 拆除、hook 否决、整体 dispose,对下游是四种完全不同的语义(要不要保留队列?要不要重放唤醒?),一个布尔全糊在一起。
2. **日志开口**。`turn/start` 落了,取消发生在 `turn/end` 之前,日志留下不闭合的括号,回放器从此要处理「悬空 turn」这种本不该存在的形态。
3. **工具幽灵**。模型发了 5 个工具调用,第 2 个执行时被取消,后 3 个的 `tool/call` 已经在模型输出里——没有对应 result 的调用会让下一次请求的历史非法(provider 会拒收孤儿 tool call)。
4. **创建期竞态**。agent 创建到一半(session 备好了、机器还没发布),此时宿主卸载/调用方取消/工厂拆除同时到达,谁负责回滚?漏一个就是泄漏,抢着回滚就是 double-free。

## 源码剖析

### 第一层:类型化的取消因由

```ts
// packages/core/agent-loop/src/agent.ts
cancel(cause: AgentCancelCause, options: CancelOptions = {}): void {
  if (!options.keepInbox) {
    this.inbox.clear()
    if (this.phase.kind !== 'idle') this.phase.wakeRequested = false
  }
  if (this.phase.kind !== 'idle') this.phase.abort.abort(cause)
}
```

`AgentCancelCause` 是闭合联合:`{ kind: 'user' } | { kind: 'parent' } | { kind: 'hook'; reason } | { kind: 'disposed' }`。它作为 abort reason 挂上信号,循环各处从 `signal.reason` 读回并按 kind 分流——最关键的分流在 `wakeDriver` 的 latch 逻辑里(第 9 章读过):`disposed` **永不 latch**,拆除中的 agent 不会因为一条迟到的消息再跑一整轮模型;其余 cause 的唤醒被闩住,等收敛后重放。`keepInbox` 则把「停掉当前动作」与「作废排队工作」解耦:web 端的 Stop 按钮用它保住用户排队的消息。

文档在这里划了一条值得记住的界线(`docs/subsystems/core.md`):cause 是 **TypeScript 强制的同进程输入**,durable 的 `turn/end` 只记粗粒度的 `{ kind: 'aborted' }`——「谁请求了取消」如果要持久化,应该是另一条独立事件,而不是把运行时对象塞进日志。live 与 durable 的分工再次出现。

### 第二层:turn() 的收敛——每条退出路径都有名字

第 8 章略过的 catch 分支,现在完整看:

```ts
// packages/core/agent-loop/src/agent.ts, turn()
} catch (error: unknown) {
  if (signal.aborted) {
    turnEnds = { kind: 'aborted', reason: signal.reason as AgentCancelCause }
    throw error
  }
  // Every failure is structured: an `LlmError` keeps its facts, anything
  // else flattens to `errorChain` text under the `UNKNOWN` code.
  turnEnds = {
    kind: 'error',
    error: error instanceof LlmError
      ? error.failure
      : { message: errorChain(error), code: 'UNKNOWN' },
  }
  this.throwError(error)
} finally {
  try {
    // oxlint-disable-next-line typescript/no-non-null-assertion -- every exit assigns a turn ending
    this.session.append('turn/end', { turn, reason: turnEnds! })
  } catch (error: unknown) {
    this.throwError(error)
  }
}
```

先判 `signal.aborted` 再判错误类型,顺序有语义:取消引发的异常(`throwIfAborted` 抛的,或流被掐断抛的)不是故障,turn 以 `aborted` 收尾并保留 cause;真正的故障则结构化——`LlmError` 携带的 provider 事实(HTTP 码、错误码、原始消息)原样入日志,杂牌异常压成 `errorChain` 文本挂 `UNKNOWN` 码。**没有任何一种异常能让 `turn/end` 不落盘**:它在 `finally` 里,而且 append 自身失败还有 `throwError` 兜底上报。

`throwError` 是「上报后再抛」的小工具:

```ts
// packages/core/agent-loop/src/agent.ts
private throwError(error: unknown): never {
  const turn = this.phase.kind === 'running' ? this.phase.turn : this.phase.lastTurn
  const step = this.phase.kind === 'running' ? this.phase.step : 0
  this.dispatch.emit('agent/error', { turn, step, error })
  throw error
}
```

verbatim 的 error 对象走实时事件 `agent/error` 给 UI/日志系统,序列化后的事实走 durable `turn/end`,原始异常继续向上传播到 `kick` 的驱动器边界被容纳(第 9 章)。一个错误,三个去向,各取所需。

LLM 流的中断则更早介入:`step()` 的 `for await` 每收一个 chunk 都 `signal.throwIfAborted()`,流中途取消时已收的 chunk 已在日志里(`assistant/chunk` 逐条 append),而 `BlockAssembler.finish` 会把中断归为 `{ kind: 'aborted' }` 交给 `agent/request-error` waterfall——所以流断裂与请求失败共用同一条恢复通道:监听器有机会 retry,否则 `LlmError` 抛出、走上面的 turn 收敛。

### 工具的收尾:合成结果堵住日志的洞

取消到达时可能有工具在跑、有工具还没起步。`tool-calls.ts` 的策略是:**已启动的排干并如实提交,未启动的补合成结果**:

```ts
// packages/core/agent-loop/src/tool-calls.ts
/** Append the durable call/result pair for a model call skipped after cancellation. */
function appendSkippedToolCall(session: Session, turn: number, step: number, block: ToolCallBlock): void {
  const callSeq = appendToolCall(session, turn, step, block)
  appendToolResult(session, turn, step, block, {
    content: [{ type: 'text', text: 'Error: tool call aborted before dispatch' }],
    isError: true,
    error: {
      message: 'tool call aborted before dispatch',
      info: { name: 'AbortError', code: TOOL_ABORTED_BEFORE_DISPATCH },
    },
  }, callSeq)
}
```

模型输出里的每个 tool-call block 都会得到配对的 `tool/call` + `tool/result`,哪怕它从未执行——模块注释说得直白:"Abort records synthetic error results for skipped calls **so replay stays valid**"。下一次请求 derive 出的历史因此永远是 provider 合法的形态,取消不会毒化后续对话。与之对照,**内部调度器故障走另一条路**:停止新派发、排干在飞的,然后带着第一个故障 reject,**不**伪造结果——环境故障不该被包装成「工具答复了一个错误」,那是对模型撒谎。两种失败,两种收尾,边界清清楚楚。

### 第三层:创建期与拆除期——index.ts 的三方熔断

现在进入 `src/index.ts`。`prepare()` 为每个新 agent 构造「机器 + scope + 一份记忆化的逆序拆除」,其中防御最重的是创建窗口:

```ts
// packages/core/agent-loop/src/index.ts, prepare()
// Deactivation fuses three owners, each with its own reason: the caller's
// cancellation signal, the owner fiber's unload, and factory teardown.
// It is registered BEFORE any resource exists, over mutable slots, so an
// unload arriving while the scope is still minting finds a working
// disposer instead of a leak.
const abort = new AbortController()
// ...caller signal 与 factory teardown 的 listener 接到 abort 上

const dispose = (ownerTriggered = false): Promise<void> => (disposing ??= (async () => {
  abort.abort(new Error(`agent "${id}" lifecycle disposed`))
  // ...移除熔断 listener
  try {
    if (machine === undefined) await machineReady.promise
    if (machine !== undefined) {
      machine.cancel({ kind: 'disposed' })
      await machine.whenIdle()
      await machine.scope.dispose()
    }
  } finally {
    try {
      detachAgent?.()
      detachSession?.()
    } finally {
      untrack()
      if (!ownerTriggered) await unfollowOwner()
    }
  }
})())
```

三个可能撕开生命周期的方向——调用方的 `signal`、owner fiber 的卸载(通过 `ownerCtx.effect` 注册的回调)、工厂整体拆除(`FactoryOwnership.signal`)——被熔进一个 `AbortController`,而且**在任何资源存在之前**注册,基于可变槽位(`machine`、`detachSession`、`detachAgent` 都是 `let`):卸载无论到得多早,拿到的都是一个能工作的 disposer,而不是泄漏。

拆除本身就是一句注释里的公式:"Disposal IS a disposed-cause cancel followed by quiescence"——`cancel({ kind: 'disposed' })` → `whenIdle()` 等驱动器收敛(`turn/end` 在这一步内落盘)→ `scope.dispose()` 回卷该 agent 的全部注册([第 11 章](11-scope.md))→ 退出注册表 → 解除记账。顺序严格逆着创建来,且 `disposing ??=` 记忆化让并发到达的三方等待同一次收敛,不存在 double-free。`publish()` 里三次 `assertLive()` 穿插在进注册表、announce、`agent/session-start` 之间——同步监听器可能当场触发拆除,每一步之后都重查存活,失败则走同一个 `dispose` 回滚。

工厂级的 `FactoryOwnership` 把这套结构再抬一层:`dispose()` 时 `accepting = false`、abort 熔断信号、并 `Promise.all` 所有在世 agent 的拆除与未完成的启动任务——插件卸载等到每个 agent 真正静默才返回。`resumeWith` 的加载路径同样被熔断覆盖:persistence 后端永不返回时,`raceAbortCall` 保证取消能赢,且迟到的 preparation 会被 `releaseAbandoned` 回调释放而不是泄漏。

### 让「恢复对了」可检查:invariant.ts

收敛逻辑这么多分支,怎么知道每条路径出来的状态都是对的?`agent-loop` 自带一个运行时断言插件:

```ts
// packages/core/agent-loop/src/invariant.ts
// Prepend prevents a short-circuiting replay listener from silencing the check.
ctx.on('llm/stream', (options: GenerateOptions, next) => {
  if (!isAgentLoopRequest(options)) return next()
  if (!Object.isFrozen(options)) fail('a loop-built request must be frozen')
  // ...session 存在性、messages 冻结、step/start 与 request/header 在日志中存在……
  const expected = session.deriveMessages()
  if (JSON.stringify(options.messages) !== JSON.stringify(expected)) {
    fail(`llm request for session "${String(session.id)}" diverges from the dispatch-time durable derivation (log-reconstruction desync)`)
  }
  // ...再比对 model/system/temperature/maxTokens/stop/tools 与折叠出的 request/header
  return next()
}, { global: true, prepend: true })
```

它挂在 `llm/stream` waterfall 的**最前端**(`prepend: true`,注释点明:防止某个短路的 replay 监听器把检查静默掉),对每一个 loop 构造的请求当场验证:请求必须冻结、必须能从日志重新推导出**逐字节相同**的 messages、必须与日志里折叠出的 `request/header` 一致。这正是取消/恢复正确性的终极裁判——无论 turn 是被取消后重开、请求失败后 retry、还是 compaction 改写了表面,只要某条恢复路径让「日志」与「即将发出的请求」脱钩,断言在**下一次请求**就爆,而不是等到 provider 拒收或用户发现答非所问。配套还有 `dsh-agent` 包的状态机断言(`packages/core/agent/src/invariant.ts`):`agent/status` 不许重复报同一状态,no-op 转换即 bug。invariant 服务本身的开关与部署策略见[第 39 章](../part10/39-invariants-and-defensive-patterns.md)。

## 设计亮点

> 💎 **设计亮点:取消因由是类型,不是布尔**
> `AgentCancelCause` 的四个 kind 各自驱动不同的收敛行为(`disposed` 不重放唤醒、`hook` 携带 reason、durable 侧统一降为 `aborted`)。普通写法的 `cancelled: boolean` 让这些分支无处安放,最终散落成一堆隐式约定;这里 kind 是 switch 得动的数据,而且「live 保留全部事实、durable 只记粗粒度结果」的分层避免了把进程内对象泄进日志。

> 💎 **设计亮点:合成 tool result——对模型诚实,对日志完整**
> 被取消跳过的工具调用补一条结构化的错误 result(带 `TOOL_ABORTED_BEFORE_DISPATCH` 码),日志括号闭合、下次请求的历史合法;而调度器自身的故障绝不伪造 result。区分「工具没能跑」与「基础设施坏了」这两种失败并给出不同的持久化策略,是这个文件最见功力的地方——多数实现要么全伪造(对模型撒谎),要么全不补(留下孤儿 call 毒化历史)。

> 💎 **设计亮点:先装 disposer,后造资源**
> `prepare()` 把三方熔断和记忆化 dispose 建立在「资源还都是 `undefined`」的时刻,靠可变槽位让 disposer 对「拆到一半的创建现场」也能正确工作。这消灭了创建期竞态的整个类别:不存在「卸载来得太早所以没东西可拆」的窗口,也不存在两个方向抢着拆的 double-free——`??=` 让所有人等同一个 promise。

> 💎 **设计亮点:恢复正确性外包给 prepend 的运行时断言**
> 与其在每条取消/重试/压缩路径里各写一遍「确保日志与请求一致」,不如在唯一的出口(`llm/stream`)装一个不可绕过的检查:`prepend: true` 保证没有监听器能抢先短路,`deriveMessages()` 逐字节比对把一切脱钩 bug 拦在发出请求之前。防御的位置选在数据流的咽喉,而不是撒遍每个分支。

## 小结与延伸

取消与恢复在这里不是散落的 try/catch,而是三层同心的收敛结构:信号层用类型化 cause 分流行为,turn 层用 catch/finally 保证日志括号必然闭合、工具层补合成结果保回放合法,生命周期层用三方熔断加记忆化拆除保证创建与销毁在任何交错下都收敛到零。最后由 prepend 的运行时断言把「一切恢复路径都没破坏日志-请求一致性」变成每个请求都被验证的硬命题。turn/step 的边界建模见[第 8 章](08-turn-and-step.md),主循环控制流见[第 9 章](09-agent-loop-deep-dive.md),"model-visible means logged" 不变量的全貌见[第 15 章](../part4/15-model-visible-invariant.md)。

**阅读清单**
- `packages/core/agent-loop/src/index.ts` — `prepare`/`FactoryOwnership`/`resumeWith` 的完整生命周期编排
- `packages/core/agent-loop/src/agent.ts` — `cancel`/`wakeDriver`/`turn` 的收敛路径
- `packages/core/agent-loop/src/tool-calls.ts` — abort 排干与合成结果的调度器实现
- `packages/core/agent-loop/src/invariant.ts` 与 `packages/core/agent/src/invariant.ts` — 两个包各自认领的运行时断言
- `docs/subsystems/core.md` 的 agent handle 与 cancellation 节 — `AgentHandle.dispose` 作为 capability 的所有权模型
- `.agents/notes/implemented/architecture/2026-07-16-explicit-turn-cancellation.md` — 显式 turn 取消的决策记录
