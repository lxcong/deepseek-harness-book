# 第 20 章 审批与权限预设

上一章的流水线在 `tools/pre-execute` 处留了一个悬念：`ask` 决策。这一章看它的落点——`packages/interaction/user-approval` 的审批座席（seam）如何让一次工具调用停下来等人、等人期间系统处于什么状态、审计如何落日志；再看 `packages/interaction/permission-presets` 如何把沙箱模式和审批策略两个旋钮打包成用户面前的一个选择器；最后补上 `packages/preset` 里保证这一切能 per-agent 组合的地基——isolate realm。需要说明：包名里带 guard 的 `packages/guard/` 装的是 timeout 和重复调用提醒（第 19 章已讲），审批本体住在 `packages/interaction/`。

## 问题背景

「危险操作先问人」听起来是一个 `if (dangerous) await confirm()`，实际要同时满足五个约束：

1. **不能开着口子失败**。审批 UI 没连上、answerer 抛异常、返回了词汇表外的值——任何一种异常都必须落向「拒绝」，而不是放行。
2. **要可审计**。谁问过、问的什么、答了什么，必须持久化，而且崩溃恢复后不能出现「问了没答」的悬案。
3. **要可组合**。CLI 场景人来答，CI 场景自动拒，ACP 自动化桥接场景机器答——同一个询问方，不同的应答方。
4. **模型要知情**。审批策略是 `ask` 还是 `never`，模型必须知道，否则它会在 CI 里反复请求一个永远不会来的授权。
5. **粒度要对**。用户不想逐个设置「沙箱开多大 × 审批问不问」，他们想要一个「安全/放开」的开关；但系统内部这两个旋钮又必须保持独立可组合。

## 源码剖析

### ask 如何短路流水线：registry 侧的 serviceAsk

回看第 19 章 `prepareExecution` 里的分支：`pre-execute` waterfall 返回 `ask` 时（比如 `hooks-claude-code` 把 hook 的 `decision === 'ask'` 直译成 `{ kind: 'ask', reason }`），registry 调 `serviceAsk`：

```ts
// packages/core/tools/src/index.ts（删节）
private async serviceAsk(exec, ask): Promise<ToolAskResolution> {
  const approval = this.ctx.get('approval')
  if (approval === undefined) {
    return { decision: { kind: 'deny', reason: ask.reason ?? `tool "${exec.name}" requires approval (not yet supported)` }, /* ... */ }
  }
  if (exec.agent === undefined) {
    return { decision: { kind: 'deny', reason: `... the call has no agent to route it through` }, /* ... */ }
  }
  const outcome = await approval.request({
    agent: exec.agent, toolName: exec.name, callId: exec.callId,
    ...ask.reason !== undefined ? { reason: ask.reason } : {},
    signal: exec.signal,
  })
  switch (outcome) {
    case 'allowed-once': return { decision: { kind: 'allow' }, approvalCancelled: false }
    case 'rejected': return { decision: { kind: 'deny', reason: `the user rejected tool "${exec.name}"` }, /* ... */ }
    // cancelled / unavailable → 各自不同 reason 的 deny
  }
}
```

注意 `ctx.get('approval')` 是**机会式消费**：文件头部专门有一行 `import type {} from '@deepseek-ai/dsh-user-approval'`——只引类型让 `ctx.get('approval')` 解析出正确的类型，运行时不依赖这个包。没组合审批服务的部署里，`ask` 直接退化为 deny；四种 outcome 里只有 `allowed-once` 是放行，且三种「不放行」各带不同的 reason 文本，模型能分辨「人拒绝了」和「根本没有审批通道」——前者该换思路，后者该停止尝试。

审批期间整个调用停在 `prepare` 阶段的 `await` 上：`tool/call` 事件已落日志、UI 已渲染 pending 卡片、工具本体尚未 dispatch。agent 的 turn 保持打开，loop 的调度器因为 pre-execute 是有序阶段而不会越过它启动后续调用。取消（用户打断 turn）通过 `signal: exec.signal` 传入，审批立刻以 `cancelled` 收场，调用落成 `ABORTED_BEFORE_DISPATCH`。

### ApprovalService.request：审计对 + 开放 turn 前置条件

```ts
// packages/interaction/user-approval/src/index.ts
async request(req: ApprovalRequest): Promise<ApprovalOutcome> {
  const session = req.agent.session
  if (!hasOpenTurn(session.events)) {
    throw new Error(
      'approval.request() outside an open turn: the approval/asked + approval/decided audit pair '
      + 'must be turn-enclosed (a bare event between turns is crash-tail garbage on reload). '
      + 'Ask from inside the turn that needs the decision.',
    )
  }
  const id = ApprovalRequestId(randomUUID())
  session.append('approval/asked', { id, toolName: req.toolName, /* callId?, reason? */ })
  const outcome = await this.decide(req, session)
  session.append('approval/decided', { id, outcome })
  return outcome
}
```

每次询问生成一个 branded `ApprovalRequestId`，把 `approval/asked` 和 `approval/decided` 配成一对 log-only 审计事件（`deriveMessages()` 不理它们，模型上下文零开销）。前置条件「必须在开放 turn 内」不是形式主义：turn 是 durable log 的提交/回放边界（[第 13 章](../part4/13-session-event-log.md)），turn 之间的裸事件在重载时和崩溃尾巴无法区分、会被静默丢弃——在 turn 外问一个「会被回放遗忘」的问题，等于制造审计假账，所以直接抛错。

`ApprovalRequest` 本身刻意**不带工具参数**：`callId` 指向已经流式展示过的那次 tool call，UI 把审批提示挂到已有卡片上，避免同一份参数渲染两份、日后漂移。

### decide：策略前置、包容异常、可撤回

```ts
// packages/interaction/user-approval/src/index.ts（删节）
private async decide(req: ApprovalRequest, session: Session): Promise<ApprovalOutcome> {
  const signal = req.signal
  if (signal?.aborted) return 'cancelled'
  // The 'never' policy is decided HERE, before any dispatch: a listener
  // registered with `prepend: true` ... cannot keep the documented promise
  // that 'never' rejects deterministically regardless of registration order
  if (this.effectivePolicy(session) === 'never') return 'rejected'
  const answer: Promise<ApprovalOutcome> = Promise.resolve().then(
    () => this.ctx.waterfall(
      scopeTarget(this, req.agent), 'approval/request', req,
      () => Promise.resolve<ApprovalOutcome>('unavailable'),
    ),
  ).then(
    outcome => OUTCOMES.includes(outcome) ? outcome : 'unavailable',
    () => 'unavailable',
  )
  if (signal === undefined) return answer
  return await new Promise<ApprovalOutcome>((resolve) => {
    const onAbort = () => { /* remove listener */ resolve('cancelled') }
    signal.addEventListener('abort', onAbort, { once: true })
    void answer.then((outcome) => { /* remove listener */ resolve(outcome) })
  })
}
```

短短四十行浓缩了四层防御。其一，`never` 策略在 waterfall **之前**由服务自己裁决——注释解释了为什么不能做成一个 waterfall listener：任何 listener 都可能被后注册的 `prepend: true` 抢到前面，「never 确定性拒绝」这个文档承诺只有服务自己的请求路径能兑现。其二，answerer waterfall 的默认值是 `'unavailable'`（没人组合 answerer 就 fail closed），web 客户端通过 apiproxy 注册 answerer，把 `approval/requested` 帧推给浏览器等人点按钮；ACP 桥接则注册机器 answerer。其三，双重归一化：同步 throw 被 `Promise.resolve().then(...)` 拉进同一条拒绝路径，异常和词汇表外的返回值都归一成 `'unavailable'`——座席包容自己的回调，绝不让 answerer 的 bug 变成放行或炸掉调用方。其四，signal race：abort 立刻以 `cancelled` 决议，晚到的答案落在已 settled 的 promise 上自然作废。

策略本身也是 event-sourced 的：`effectiveApprovalPolicy(events)` 从日志尾部往前找最后一条 `approval/policy` 事件，「回放日志就是状态」，resume 不需要任何追赶机制。模型侧的知情通过 runtime context 实现——服务构造时注册了一个 `approval:policy` context（order 115），按当前策略渲染 `NEVER_SENTENCE` 或 `ASK_SENTENCE`；放在动态 context 而不是 system prompt 里，是为了切换策略时不改写请求头的稳定前缀（prompt cache 友好，细节见[第 21 章](21-system-prompt.md)）。

### Permission presets：两个旋钮，一个选择器

`packages/interaction/permission-presets` 把 `sandbox/mode` 和 `approval/policy` 两个独立旋钮打包：

```ts
// packages/interaction/permission-presets/src/index.ts
export interface PresetSpec {
  sandbox: SandboxMode      // 'workspace-write' | 'danger-full-access' | ...
  approval: ApprovalPolicy  // 'ask' | 'never'
  name?: string
  description?: string
}

// 默认表：
// 'workspace-write'    → { sandbox: 'workspace-write',    approval: 'ask' }
// 'danger-full-access' → { sandbox: 'danger-full-access', approval: 'never' }
```

关键在写路径 `apply`：preset 切换**不是**一个新的状态源，而是一次「记录意图 + 逐旋钮写穿」：

```ts
// packages/interaction/permission-presets/src/index.ts
private apply(session: Session, name: string, setApproval: (policy: ApprovalPolicy) => void): void {
  const spec = this.resolve(name)
  if (this.current(session.events) !== name) {
    session.append('permission/preset', { preset: name })   // log-only 用户意图
  }
  const events = session.events
  if (spec.sandbox !== (effectiveSandboxMode(events) ?? this.ctx.shell.sandboxMode)) {
    setSandboxMode(session, spec.sandbox)                   // canonical setter
  }
  if (spec.approval !== (effectiveApprovalPolicy(events) ?? this.ctx.approval.config.policy ?? 'ask')) {
    setApproval(spec.approval)                              // canonical setter
  }
}
```

执行、prompt 叙述、回放继续只读各自旋钮的 fold（`sandbox/mode`、`approval/policy` 事件），preset 层对它们透明；`permission/preset` 事件只保存「用户选了哪个名字」——它存在的理由很微妙：两个 preset 可能捆绑同一组旋钮值，反推会歧义，`derive()` 里「上次选择若仍匹配则优先」的规则靠它兑现。旋钮值匹配不到任何表项时报告为保留名 `custom`（不可作为切换目标，构造时禁止表里出现这个名字）。读侧是 `permissions` session projection（老 UI 概念里的下拉框数据），写侧是 `/permission` 命令——两者都是**可选 child**，用 `ctx.inject(['sessionProjections'], ...)`／`ctx.inject(['commands'], ...)` 挂载，headless 部署没有这两个注册表时服务照常工作。

构造函数里还有一条硬校验：`ctx.shell.sandboxMode === undefined`（挂了不限制的 executor）直接抛错——preset 捆绑沙箱模式，把它组合在不设限的 shell 上是配置错误，fail loud。

### preset 包的地基：isolate realm

per-agent 的审批/权限组合能成立,前提是 agent preset 里 mount 的服务不会泄漏成进程全局。`packages/preset/agent-presets/src/mount.ts` 在 mount 完成时做泄漏检查：

```ts
// packages/preset/agent-presets/src/mount.ts（删节）
const leaked = leakedServices(agentCtx, fiber)
if (leaked.length > 0) {
  throw new Error(
    `row(s) published process-global service(s) [${leaked.join(', ')}]; `
    + 'a preset service must sit behind an `isolate` realm or move to the host composition',
  )
}
```

`isolate` realm 是 Cordis 的机制：服务实现默认存在 root realm 的 symbol 下（进程内全局可见），而放进 isolate realm 的服务存在 realm 私有 symbol 下，组外任何 context——包括宿主——都解析不到。`leakedServices` 的判定就是这个存储事实的直接读取：遍历 service store 的 symbol，凡属于被 mount 子树、又被 root realm 的 symbol 表指到的，就是泄漏。两个 session 各自 mount 同一个 preset 时，各自的服务实例互不碰撞，靠的正是这道构造性隔离；而当一个外部调用者（浏览器 RPC）确实需要读某个 agent 的 realm 私有服务时，`serviceForAgent` 提供只读寻址——沿 agent 的 scope 链找到 standing mount，再按 fiber 归属反查实例。隔离是默认,穿透是显式且只读的。

## 设计亮点

> 💎 **设计亮点：审批是可选座席，缺席即拒绝。** 普通写法要么把审批做成硬依赖（headless 部署被迫拖着 UI 包），要么在调用点写 `if (approvalService)` 并在缺席时放行（安全事故）。这里 `ctx.get('approval')` + type-only import 让依赖在运行时可选、在类型上精确，且缺席的退化方向被钉死在 deny——四种 outcome 只有一种放行，异常、rogue 返回值、缺席全部归一到不放行的一侧。

> 💎 **设计亮点：审计对必须被 turn 包住。** `approval/asked`/`decided` 若允许落在 turn 之间，崩溃恢复时会被当作 crash tail 丢弃，审计出现「问过但没有记录」的洞。`request()` 用 `hasOpenTurn` 把这个回放语义的坑变成一条会抛错的前置条件，错误信息直接把原因和正确做法写给你看。把不变量放在离违规最近的地方检查，是这个仓库反复出现的手法（[第 39 章](../part10/39-invariants-and-defensive-patterns.md)）。

> 💎 **设计亮点：`never` 策略住在服务里，不住在 waterfall 里。** 「never 必然拒绝」是文档承诺，而 listener 顺序是运行时可变的（`prepend: true` 可以插队）。把这条裁决放进 `decide()` 的请求路径而非注册一个 gate listener，是对「扩展点的灵活性」和「承诺的确定性」边界的清醒认识：可协商的进 waterfall，不可协商的进代码。

> 💎 **设计亮点：preset 记录意图，旋钮保持权威。** `permission/preset` 事件不驱动任何执行语义——沙箱和审批各自的 fold 才是权威，preset 只是写穿两个 canonical setter 并留下「用户选了这个名字」的意图记录，用于共享捆绑时的反推去歧义。加一层用户体验抽象而不新增状态源，回放、执行、UI 三方都不需要知道 preset 的存在。

## 小结与延伸

审批是一条挂在 `tools/pre-execute` 上的可选座席：`ask` 让调用停在 prepare 阶段等待，turn 保持打开，audit 对落日志，answerer 链 fail closed，signal 可随时撤回。权限预设在沙箱模式与审批策略之上加了一层纯意图的打包，而 agent-presets 的 isolate realm 检查保证这些 per-agent 组合不会互相踩踏。三个包合起来回答了「谁可以做什么」——下一章回答「模型被告知了什么」。

延伸阅读：

- `docs/subsystems/approval.md`、`docs/subsystems/permission-presets.md` — 官方文档
- `packages/interaction/user-approval/src/types.ts` — branded id 与 closed outcome 词汇表
- `packages/host/apiproxy/src/api/events.ts` — `approval/requested`/`approval/resolved` 如何推到浏览器
- `packages/shell/tool-bash/src/index.ts` — 沙箱升级（`sandbox_permissions`）这个第二个审批消费方
- [第 30 章](../part8/30-subagent.md) — 委托时 `approval/policy` 如何以 `source: 'delegation'` 种进子 session
