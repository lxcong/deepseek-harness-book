# 第 9 章 agent-loop 源码精读:主循环的七百行

`packages/core/agent-loop` 是整个 harness 的心脏,但它的七百行拆在两处:`src/index.ts`(713 行)是工厂与生命周期插件——创建、发布、有序拆除每个 agent;真正的驱动循环在 `src/agent.ts`(496 行)的 `ReactLoopAgent` 里。本章沿着一次完整的驱动流走读后者的主干:claim 输入 → `agent/pre-step` waterfall → 组装 prompt/tools → `agent/request` waterfall → `llm/stream` → 工具执行 → `step/end` → 续步判定,并挑出四段最能体现控制流设计的代码细讲。`index.ts` 的生命周期与拆除路径留给[第 12 章](12-cancellation-and-recovery.md)。

## 问题背景

自己写这个循环,朴素版大概是:一个 `async run()` 方法,里面 `while` 套 `while`,请求参数现场拼,消息历史存在成员变量数组里,取消靠一个 `this.stopped` 布尔。它会在四个地方翻车:

1. **并发唤醒**。两条消息几乎同时到达,两个 `run()` 跑起来,同一个 session 被写花。你需要「同一时刻至多一个驱动器」的排它保证,而且不能用锁——Node 里锁不住 await 之间的空隙。
2. **扩展点缺失**。想在模型看到消息前改写它?想换模型路由?想在请求失败后按策略重试?朴素实现只能往循环里塞 if,每加一个需求改一次核心代码。
3. **请求与历史脱钩**。成员变量里的 `messages` 数组和持久化日志各自演化,一次 resume 之后两边对不上,模型看到的和日志记的不再是同一份现实。
4. **重入礼仪**。事件监听器在回调里同步调用 `cancel()` 或再发消息,朴素实现的状态在自己脚下被换掉。

`ReactLoopAgent` 对这四个问题的回答分别是:phase 状态机 + promise 记账的单驱动器保证、四个 Cordis 事件扩展点(`agent/pre-step`、`agent/request`、`agent/request-error`、`agent/turn-stopping`)、**每次请求都从日志重新推导**(derive from log)、以及处处显式的 `signal.throwIfAborted()` 检查点。

## 源码剖析

先给全景。一次驱动流按调用层级展开:

```text
send(wakeup=true) → wakeDriver() ─ 占据 running phase
  └ kick() ─ 驱动器边界,包含所有异常
      └ while (await this.turn()) {}
          └ turn() ─ 开/关 turn(第 8 章)
              ├ preStep(target) ─ claim + prompt 组装 + pre-step waterfall
              └ step(assembly)
                  ├ buildRequest() ─ agent/request waterfall + request/header 落日志
                  ├ llm.stream() ─ 逐 chunk 落 assistant/chunk
                  ├ 失败 → agent/request-error waterfall(retry 或抛出)
                  ├ assistant/message 落日志
                  └ executeToolCalls() ─ tool/call + tool/result 落日志
```

### 片段一:wakeDriver + kick——不用锁的单驱动器保证

```ts
// packages/core/agent-loop/src/agent.ts
private wakeDriver(wakeAfterAbort = false): void {
  if (this.phase.kind !== 'idle') {
    // Maintenance and aborted drivers cannot deliver the wake: latch it for
    // replay at convergence. Live drivers claim queued work themselves;
    // disposal never latches, so teardown waits on no model turn.
    const reason = this.phase.abort.signal.reason as AgentCancelCause | undefined
    if (reason?.kind !== 'disposed' && (this.phase.kind === 'maintenance' || wakeAfterAbort)) {
      this.phase.wakeRequested = true
    }
    return
  }
  const driver = Promise.withResolvers<void>()
  this.activityDone = driver.promise
  this.setPhase({
    kind: 'running',
    abort: new AbortController(),
    turn: this.phase.lastTurn,
    step: 0,
    wakeRequested: false,
  })
  this.loopCtx.agents.withInitiator(this, () => this.kick()).then(driver.resolve, driver.reject)
}

private async kick(): Promise<void> {
  try {
    while (await this.turn()) {}
  } catch (_error) {
    // Reported failures and cancellation are contained at the driver boundary.
  } finally {
    if (this.phase.kind === 'running') {
      const { turn, wakeRequested } = this.phase
      this.setPhase({ kind: 'idle', lastTurn: turn })
      if (wakeRequested && this.inbox.hasPending) this.wakeDriver()
    }
  }
}
```

排它性的实现完全是同步的:`wakeDriver()` 从头到 `setPhase(running)` 之间没有任何 await,JS 单线程保证两次唤醒不可能都看到 `idle`——第二次进来时 phase 已是 `running`,直接走顶部的早退分支。不需要锁,因为**临界区被刻意压缩成同步代码**。

早退分支里的 latch(`wakeRequested = true`)处理一个精细的时序窗口:活着的驱动器自己会去 claim 队列,不需要 latch;但 maintenance 任务不读队列,被 abort 的驱动器正在收敛也无法交付,这两种情况就把唤醒「闩」起来,由退出中的活动在自己的 `finally` 里重放(`kick` 尾部那句 `if (wakeRequested && this.inbox.hasPending) this.wakeDriver()`)。重放放在 `finally` 里而不是用 `activityDone.then(...)` 链,保证 `turn/end N` 一定先于重放驱动器的 `turn/start N+1` 落日志——仓库的 Agent Note(`2026-08-07-cancel-convergence-wake-latch.md`)记录了这个决策淘汰掉的三个替代方案。

`kick` 的空 catch 不是吞错:错误在抛到这里之前已经由 `throwError()` 通过 `agent/error` 事件上报、由 `turn/end` 以 `{ kind: 'error' }` 持久化(见[第 12 章](12-cancellation-and-recovery.md)),驱动器边界只负责「无论如何回到 idle」。还有一处容易錯過:`withInitiator(this, ...)` 把整个驱动器包在 initiator scope 里,后面 `executeToolCalls` 里的 `ctx.agents.requireInitiator()` 靠它拿到发起 agent——工具执行代码因此不需要把 agent 一层层传下去。

### 片段二:preStep——claim、组装与 waterfall 的合流

```ts
// packages/core/agent-loop/src/agent.ts
private async preStep(target: InboxTarget, position: { turn: number; step: number }): Promise<PreparedStep> {
  // ...
  const signal = this.phase.abort.signal
  const claimed = this.inbox.claim(target, position.turn)
  const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))
  signal.throwIfAborted()
  const sections = renderContextSections(assembly)
  const context = this.runtimeContext.project(joinContextSections(sections), sections)
  const decision = await this.dispatch.waterfall(
    'agent/pre-step', { messages: claimed, ...position, signal },
    (): Promise<PreStepDecision> => Promise.resolve<PreStepDecision>({
      kind: 'enter',
      messages: context === undefined ? claimed : [...claimed, context],
    }),
  )
  signal.throwIfAborted()
  return decision.kind === 'reject' ? decision : { ...decision, assembly }
}
```

十几行做了四件事,顺序都有讲究。先 `claim`:这批消息从此归这个 turn 所有,即使后面被 reject 也不退回(第 8 章讲过零 step turn)。再 `assemble`:system prompt 的 sections 和工具 schema 在**每个 step**重新组装——插件热插拔的工具、随环境变化的动态 context,都在下一个 step 自然生效,没有缓存失效问题要处理。

最妙的是 waterfall 的默认值写法。Cordis waterfall 的最后一个参数是「最内层的 `next()`」——当没有监听器,或所有监听器都调用了 `next()` 时执行的兜底逻辑。这里的兜底是「enter,消息为 claimed 批次,外加可能的 runtime context 快照」。于是插件的三种姿态自然分层:不注册监听 = 默认行为;注册并调 `next()` 再修剪返回值 = 包装式改写(compaction 的裁剪就这么做);不调 `next()` 直接返回 = 整体接管(reject 或替换)。核心循环完全不知道有没有插件存在。

`runtimeContext.project(...)` 那行是动态上下文的去重投影:`RuntimeContextProjection`(`src/runtime-context.ts`)追踪日志里最后一条保留的 runtime-context 快照,只有内容变了才生成新的 `UserMessage` 搭进本 step,内容清空时则投一条 "Current runtime context: none" 的显式失效标记——避免每个 step 都重复注入相同的环境描述。

### 片段三:step——流式循环与结构化重试

```ts
// packages/core/agent-loop/src/agent.ts
private async step(assembly: PromptAssembly): Promise<StepEndReason | null> {
  // ...
  while (true) {
    const { request, preparedCall } = await this.buildRequest(
      turn, step, assembly.tools, system, this.session.deriveMessages(), signal,
    )
    const assembler = new BlockAssembler()
    const chunkSeqs: number[] = []
    const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
    signal.throwIfAborted()
    for await (const chunk of stream) {
      signal.throwIfAborted()
      chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
      assembler.push(chunk)
    }
    signal.throwIfAborted()
    const finish = assembler.finish
    if (finish.kind === 'error' || finish.kind === 'aborted') {
      const action = await this.dispatch.waterfall(
        'agent/request-error', { turn, step, provider: request.provider,
          failure: finish.failure, retryPolicy: preparedCall?.retryPolicy, signal },
        () => Promise.resolve<RequestErrorAction>(undefined),
      )
      signal.throwIfAborted()
      if (action?.kind !== 'retry') {
        throw new LlmError(finish.failure.message, finish.failure.code, finish.failure)
      }
      continue
    }
    // ...assistant/message 落日志(带 usage 与 chunkSeqs 溯源)
    if (finish.kind === 'max-tokens') return { kind: 'max-tokens' }
    const toolCalls = message.content.filter(block => block.type === 'tool-call')
    if (toolCalls.length === 0) return { kind: 'completed' }
    const { concluded } = await executeToolCalls(
      this.loopCtx, turn, step, toolCalls, signal,
      context => this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context]),
    )
    return concluded ? { kind: 'completed' } : null
  }
}
```

注意 `buildRequest` 的第五个实参:`this.session.deriveMessages()`。**模型历史不存在任何成员变量里**——每次请求都从 append-only 日志现场投影。上一个 step 刚 append 的 `assistant/message` 和 `tool/result`,到这一步通过投影自然出现在历史里;fork、resume、compaction 改写的是日志,循环无需知道。配套的运行时断言(`src/invariant.ts`,第 12 章细讲)会在 `llm/stream` 边界校验请求里的 messages 与 dispatch 时刻的 `deriveMessages()` 逐字节一致,把「日志-请求脱钩」这类 bug 钉在发生现场。

流式循环里每个 chunk 先 `append('assistant/chunk')` 再喂给 assembler,并把返回的 `seq` 收进 `chunkSeqs`——最终 `assistant/message` 事件的 `sourceEventSeqs` 会引用这些 chunk 事件,日志内部形成「聚合事实引用原始事实」的溯源链。

失败路径同样走 waterfall:默认值是 `undefined`(失败即终局),监听器返回 `{ kind: 'retry' }` 则 `continue` 重新走一遍 `buildRequest`——注意重试会**重新推导历史、重新过 `agent/request` waterfall**,所以 compaction 插件可以在 `agent/request-error` 里裁剪日志表面,然后让重试自然带上瘦身后的上下文。这就是「错误恢复不是特殊路径,而是再走一次正常路径」。

`step()` 的返回值设计也值得一提:`StepEndReason | null`,其中 `null` 意味着「工具执行完但账没清」(有 tool result 且无 `concludesTurn`),由 `turn()` 判定续步。工具结果的 `additionalContexts` 通过回调 splice 进 `next-step` 队列,和用户 steering 走完全相同的入口——循环里不存在「工具上下文」的专用通道。

### 片段四:turn() 尾部——续步判定与 turn-stopping 的双重检查

```ts
// packages/core/agent-loop/src/agent.ts, turn() 内层循环尾部
      signal.throwIfAborted()
      if (turnEnds && this.inbox.nextStep.length === 0) {
        await this.dispatch.serial('agent/turn-stopping', { turn, signal })
        signal.throwIfAborted()
      }
      if (turnEnds && this.inbox.nextStep.length === 0) break
      target = 'next-step'
```

同一个条件写了两遍,不是手误。`agent/turn-stopping` 是 serial 事件(逐个 await,无 `next()`),监听器如果反对关 turn,做法不是返回什么值,而是**调 `agent.steer(...)` 往 next-step 队列塞消息**;await 回来后第二次检查同一条件,队列非空就 `target = 'next-step'` 继续循环。协议用数据说话:监听器的注册顺序、返回值都影响不了结局,唯一有效的表达方式是往队列里放东西。第 8 章引过文档的总结——"Data decides, so listener order cannot change the outcome"。

跳出内层循环后,`finally` 落 `turn/end`,然后是驱动器级的续 turn 判定:

```ts
// packages/core/agent-loop/src/agent.ts, turn() 末尾
  if (!this.inbox.hasPending) return false
  phase.abort = new AbortController()
  // A fresh controller makes a latch set on the old one stale: the live driver claims the queue itself.
  phase.wakeRequested = false
  phase.step = 0
  return true
```

队列里还有排队的 follow-up,就换一个新的 `AbortController`(上一个 turn 的取消不应波及下一个 turn)、清零 step 计数,返回 `true` 让 `kick` 的 `while` 续开下一个 turn——这就是「一次唤醒消化多个 turn」的实现位置。

## 设计亮点

> 💎 **设计亮点:请求即投影——循环不持有任何对话状态**
> `ReactLoopAgent` 的成员变量里没有 messages 数组。每个请求的历史来自 `session.deriveMessages()`,每个请求的配置从日志里折叠的 `request/header` 出发(`buildRequest` 里的 `requestProposal(persistedHeader)`)。换普通写法——内存数组 + 日志双写——resume/fork/compaction 每一个都要写状态同步代码,而且同步 bug 只会在离现场很远的地方爆炸。这里日志是唯一事实源,循环是它的纯函数消费者,还有 invariant 在运行时逐请求验证这一点。

> 💎 **设计亮点:waterfall 默认值 = 内联的核心行为**
> 四个扩展点(`pre-step`、`request`、`request-error`、`turn-stopping` 前三个)都把「无插件时的行为」写成 waterfall 的最内层 `next()`。核心循环因此不需要 `if (hasPlugins)` 分支,插件也拿到了完整的三态表达力:透传、包装、接管。对比常见的「before/after 钩子对」设计:钩子改不了核心行为本身,而 waterfall 里核心行为只是链条的最后一环。

> 💎 **设计亮点:同步临界区替代锁**
> 单驱动器保证不靠互斥量,靠「检查 phase 到占据 phase 之间零 await」的书写纪律;唤醒丢失窗口不靠轮询,靠 phase 上的 `wakeRequested` latch 在退出活动的 `finally` 里重放。整套并发控制没有引入任何同步原语,全部是对 JS 单线程执行模型的精确利用——代价是这些代码段的 await 位置成为契约的一部分,仓库用注释把每一处「为什么此处不能 await」都写明了。

> 💎 **设计亮点:重试 = 重新走正常路径**
> `agent/request-error` 返回 retry 后只是 `continue`:重新 derive 历史、重新过 request waterfall、重新流式。恢复插件(如 compaction)只需在自己的监听器里修好持久状态,不需要理解循环内部;循环也不为恢复留任何特殊状态。错误路径与正常路径共用同一段代码,意味着错误路径被每一次正常请求「测试」着。

## 小结与延伸

`agent-loop` 的主循环把控制流压进四层结构:`wakeDriver` 用同步临界区保证单驱动器,`kick` 是异常与状态收敛的边界,`turn()` 管边界事件与续步判定,`step()` 管一次请求的完整生命周期。贯穿始终的两条纪律——一切模型可见内容先落日志、一切扩展通过 waterfall 默认值内联——让这七百行既是核心实现,也是插件生态的地基。工具调度器 `tool-calls.ts` 的 barrier/滚动池细节见[第 19 章](../part5/19-tool-pipeline.md),`deriveMessages()` 投影规则见[第 14 章](../part4/14-derive-messages.md),`buildRequest` 里 request/header 的折叠与变更追踪同章展开。

**阅读清单**
- `packages/core/agent-loop/src/agent.ts` — 本章主角,建议按 `wakeDriver → kick → turn → preStep → step → buildRequest` 顺序通读
- `packages/core/agent-loop/src/tool-calls.ts` — `executeToolCalls` 的模型序提交与并行池
- `packages/core/agent/src/dispatch.ts` — `agentEvents` fused dispatcher:emit 的逐监听器容错、waterfall 的类型融合
- `.agents/notes/implemented/bug-fix/2026-08-07-cancel-convergence-wake-latch.md` — wake latch 的完整决策记录,含被否决的三个方案
- `docs/agent-lifecycle.md` — 与本章控制流对应的事件时序图
