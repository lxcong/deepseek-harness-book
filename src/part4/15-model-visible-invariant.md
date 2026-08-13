# 第 15 章 "Model-visible means logged"：一条运行时不变量

`docs/architecture.md` 的 Session log 一节用加粗字体写了一句话：**"Model-visible means logged."**——凡是到达模型请求的内容，必须能从日志重建，并且有一条 runtime invariant 在断言它。本章找到这条不变量的实现代码（`packages/core/agent-loop/src/invariant.ts`），看它如何在每次 LLM 请求发出前把请求体与日志投影逐字节对账，然后退一层看承载它的 invariants 基础设施：为什么每个包都要有一个 `./invariant` 伴生插件，以及这套纪律如何把「上下文漂移」这类最难调试的 bug 变成当场爆炸的断言失败。

## 问题背景：上下文漂移是最脏的一类 bug

Agent harness 里插件众多，每个都可能想「顺手」影响模型看到的内容：一个插件在发请求前往 messages 里塞一条提示、另一个直接改了 system prompt 字符串、第三个在内存里维护了一份「修正过的」history。短期一切正常——直到：

- 用户 resume 会话，重放日志得到的 history 里**没有**那条内存中塞入的消息，模型行为突变；
- 排查线上问题时想知道「那次请求模型到底看到了什么」，日志给不出答案；
- 两个插件各改各的，叠加顺序在一次重构后悄悄变化。

这类 bug 的共同点是：**写入点不唯一**。日志说一套，请求说另一套，而且分叉发生在几百个 step 之前，等症状出现时现场早没了。传统的应对是 code review 纪律（「请通过 X API 注入上下文」），但纪律不可执行。DeepSeek Harness 把纪律变成断言：请求与日志投影不一致，当场抛错。

## 源码剖析

### 不变量本体：请求 = 日志投影

`packages/core/agent-loop/src/invariant.ts` 全文只有 60 行，核心是挂在 `llm/stream` waterfall 上的一个检查：

```ts
// packages/core/agent-loop/src/invariant.ts
const install: InvariantInstaller = Object.assign((ctx: Context, fail: InvariantFailure) => {
  // Prepend prevents a short-circuiting replay listener from silencing the check.
  ctx.on('llm/stream', (options: GenerateOptions, next) => {
    if (!isAgentLoopRequest(options)) return next()
    if (!Object.isFrozen(options)) fail('a loop-built request must be frozen')
    if (options.sessionId === undefined) fail('a loop-built request must carry a session id')
    const session = ctx.sessions.get(options.sessionId)
    if (!session) fail(`a loop-built request must carry a live session id, ...`)
    // ...
    const events = session.events
    if (!events.some(event => event.type === 'step/start')) {
      return fail('a loop-built request with no step/start in its session log')
    }
    const header = foldRequestHeader(events)
    if (header === undefined) {
      return fail('a loop-built request with no request/header event in its session log')
    }
    const expected = session.deriveMessages()
    if (JSON.stringify(options.messages) !== JSON.stringify(expected)) {
      fail(`llm request for session "${String(session.id)}" diverges from the dispatch-time durable derivation (log-reconstruction desync)`)
    }

    const headerMatches = options.model === header.config.model
      && options.system === header.system
      && options.temperature === header.config.temperature
      && options.maxTokens === header.config.maxTokens
      && JSON.stringify(options.stop) === JSON.stringify(header.config.stop)
      && JSON.stringify(options.tools ?? []) === JSON.stringify(header.tools ?? [])
    if (!headerMatches) {
      fail(`llm request for session "..." diverges from the folded request header`)
    }
    return next()
  }, { global: true, prepend: true })
}, { inject: ['sessions'] })
```

对账分两半，覆盖了模型可见面的全部：

1. **消息**：`options.messages` 与 `session.deriveMessages()` 做 `JSON.stringify` 整体比对。这里复用的正是第 14 章那条唯一投影规则——invariant 不需要自己理解 surface 语义，它只问「你发出去的，和从日志推出来的，是不是同一个东西」。任何绕过日志改 messages 的插件立刻触发 `log-reconstruction desync`。
2. **请求头**：model、system prompt、temperature、maxTokens、stop、tools schema，逐一与 `foldRequestHeader(events)` 的折叠结果比对。这就是为什么第 13 章的事件词汇里有 `request/header`——system prompt 和工具表也是 model-visible 的，所以它们也必须入日志（agent loop 在每个 step 构建请求时，头有变化就追加 `request/header` 快照，见 `agent.ts` 的 `buildRequest`）。

两个细节决定了这条检查「不可绕过」。其一，`prepend: true`：`llm/stream` 是 waterfall（第 5 章），后注册的 listener 可以短路不调 `next()`——比如一个 replay/mock 插件直接返回录制的响应。prepend 保证 invariant 排在所有人前面，谁也没机会在它之前拦截请求。其二，`if (!Object.isFrozen(options)) fail(...)`：请求对象必须冻结，检查过后也没有人能再改——对账结果在发出瞬间仍然成立。

### 推论：新的 model-visible 输入必须是新事件

这条不变量的存在把一个架构规则变成了物理定律。`docs/architecture.md` 写道：

> **Model-visible means logged.** Anything that reaches a model request must be reconstructable from the log, and a runtime invariant asserts it. This is why a new model-visible input requires a new session event: extend `SessionEventMap` and render from the log.

想给模型注入新内容？你**没有**别的路：直接改 `options.messages` 会被 invariant 当场击毙，所以唯一可行的实现是——merge 一个新事件类型进 `SessionEventMap`（或复用 `user/message` 带上你的 `source`），把内容 append 进日志，让 `deriveMessages()` 自然把它投影出来。仓库里的实际扩展都是这么做的：inbox 注入的上下文（文件变更通知、skill 内容、cron 提醒）最终都落成带特定 `source` 的 `user/message` 事件；压缩摘要（第 16 章）落成 `surfaceOp: replace` 的 `user/message`。于是 resume、fork、审计、telemetry 自动覆盖每一种新输入——因为它们读的就是日志。

上下文漂移在结构上被消灭：不是「大家小心别漂移」，而是「漂移的代码活不过第一次请求」。

### 基础设施：包自有的 invariant 伴生插件

这条检查不是孤例。`docs/subsystems/invariants.md` 描述的 `ctx.invariants` 是一个注册表服务（`packages/runtime-diagnostics/invariants`）：**每个 workspace 包**都要发布一个 `./invariant` 伴生插件，以自己的 npm 包名注册检查：

```ts
// packages/core/agent-loop/src/invariant.ts
export const apply = (ctx: Context): Promise<() => void> =>
  Promise.resolve(ctx.invariants.register(PACKAGE_NAME, install))
```

`fail(message)` 抛出的 `InvariantError` 带着注册包名，消息前缀是 `invariant violated by "<package>": ...`——违规可归因，注册表本身不 import 任何产品包。选择哪些包的检查生效由配置的 regex allowlist/blocklist 决定（生产可关，测试全开）。而且注册即占名：两个插件不可能静默认领同一个包名。

约定里最有分寸的一条是「assertions are deliberately not synthetic」：一个包只有当它拥有可观察的事件流或可变数据关系时才装检查，否则导出一个空 installer，并用 `No runtime invariant:` 开头的注释解释**为什么**没有可断言的东西——`pnpm run verify-package-invariants` 会机械拒绝没解释的空 installer、忽略 `fail` 参数的假检查、注错名的注册。连「没有检查」都要过 CI。

### session 包自己的 invariant：关系校验的 staged transition

作为对照，看 session 包的伴生检查（`packages/core/session/src/invariant.ts`）。它断言的是日志内部的**关系**不变量：seq 严格递增、turn/step 正确配对嵌套、`tool/result` 必须有同 step 的前置 `tool/call`……有意思的是它的接线方式：

```ts
// packages/core/session/src/invariant.ts
ctx.on('internal/dispatch', (_mode, eventName, args) => {
  if (eventName !== 'session/event') return
  const [session, event] = args as [Session, SessionEvent]
  const trace = traceFor(session)
  const transition = validateEvent(trace, event, fail)
  // A later dispatch listener may veto. Validation is pure, so abandoning
  // this weakly keyed transition does not advance or retain the session.
  stagedTransitions.set(event, { session, trace, transition })
}, { global: true })

ctx.on('session/event', (session, event) => {
  const staged = stagedTransitions.get(event)
  if (staged === undefined || staged.session !== session) {
    return fail('session/event reached publication without matching pre-commit validation')
  }
  stagedTransitions.delete(event)
  applyTransition(staged.trace, staged.transition)
}, { global: true })
```

回忆第 13 章 `append()` 的顺序：listener 快照在 push 前解析（此时 `internal/dispatch` 触发），回调在 push 后执行（此时 `session/event` 触发）。这个 invariant 恰好骑在这两个时刻上：**验证发生在事件进日志之前**（一个违规事件在 `fail` 抛出时还没提交，Cordis 的 dispatch 校验失败会让 append 整体拒绝），**状态推进发生在提交之后**（append 若被其他原因拒绝，staged transition 被 WeakMap 悄悄回收，checker 自己的 trace 不会错误前进）。检查器与被检查系统之间做到了事务级同步——大多数项目的「校验中间件」连自己的状态一致性都保不住。

> 💎 **设计亮点：断言关系，而不是重新实现**。这条 invariant 没有第二套「正确的请求构建逻辑」拿来对照——那只会造出两份会各自烂掉的实现。它对账的两边都是生产代码自己的产物：`options.messages` 对 `session.deriveMessages()`，请求头对 `foldRequestHeader(events)`。invariant 的成本只是一次序列化比较，而覆盖面是「任何来源的任何偏差」。

> 💎 **设计亮点：prepend 让检查不可静默**。断言最怕被绕过。`llm/stream` 是可短路的 waterfall，一个 mock/replay 插件完全可能在生产配置里存在。`prepend: true` 一个词把检查钉在链条最前端——这是对自家事件系统语义（第 5 章）的精确运用：知道哪里可能被短路，就把自己放到短路点之前。

> 💎 **设计亮点：「没有检查」也要过验证**。`verify-package-invariants` 机械强制每个包要么有真检查、要么有一段解释为什么无可检查。普通项目的 invariant 覆盖靠热情，热情随人员流动衰减；这里把覆盖率做成了 CI 里的结构性事实，而「宁可解释也不许假装」的规则挡住了为过检而写的 synthetic assertion。

## 小结与延伸

"Model-visible means logged" 是一条三层结构的纪律：架构文档陈述它，`SessionEventMap` 的扩展机制给出唯一合规实现路径，`agent-loop/src/invariant.ts` 在每次请求前用 `deriveMessages()` 与 `foldRequestHeader()` 对账强制它。承载它的 invariants 服务把「包自有、可归因、pre-commit 验证」做成了全仓库的通用设施。上下文漂移——多写入点系统里最难缠的一类 bug——从「靠 review 防」变成「结构上不可能存活」。下一章看这条日志纪律兑现回报的地方：fork、resume 与 compaction 全都只是对同一条日志的重放与改写。

**阅读清单**

- `packages/core/agent-loop/src/invariant.ts` — 本章主角，60 行全文值得通读
- `packages/core/session/src/invariant.ts` — 关系校验与 staged transition
- `packages/runtime-diagnostics/invariants/src/index.ts` — 注册表服务
- `docs/subsystems/invariants.md`、`docs/architecture.md`（Session log 节）
- [第 14 章](14-derive-messages.md) — 被对账的投影 `deriveMessages()`
- [第 39 章](../part10/39-invariants-and-defensive-patterns.md) — 全仓库 invariant 文化的横向盘点
