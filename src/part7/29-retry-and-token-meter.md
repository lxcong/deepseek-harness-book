# 重试与 token-meter

模型层的最后两块拼图都是「旁观者」：`llm-retry` 在请求失败后决定要不要再来一次，`token-meter` 随时回答「这个会话现在有多重」。两者有一个共同的、也是本章最想讲清楚的设计选择——它们都**不**站在请求的必经之路上，而是完全建立在 session log 之上。先纠正一个想当然的预设：验证源码后可以确认，`llm-retry` 并不是包裹 `llm/stream` 的 waterfall 中间件，它挂在 agent loop 的 `agent/request-error` 恢复扩展点上；token-meter 计量的也只是 token（含缓存分桶），仓库里没有任何货币成本换算。

## 问题背景

朴素的重试写在 HTTP client 里：`fetch` 失败就 sleep 后再打。三个问题立刻出现。其一，不可见——用户盯着卡住的界面，日志里什么都没有，进程一崩连「刚才在等第几次重试」都无从谈起。其二，位置错——在 `llm/stream` 内部重试意味着已经 yield 给消费方的半截 chunk 流没法收回，只能在流中间偷偷「重新开始」，下游的拼装器和日志全被污染。其三，策略与执行耦合——每家 provider 的可重试错误码、退避参数都不同，写死在执行代码里就要为每家改核心。

朴素的 token 计量则是每次渲染状态栏时对全量上下文跑一遍 tokenizer——贵，且对不上 provider 的真实计费口径（缓存命中、CJK 密度、JSON schema 开销）。DeepSeek Harness 对这两个问题给出同构的答案：**把状态放进日志，把计算做成 fold**。

## 源码剖析

### 重试的三权分立：policy、闭步、executor

重试被拆成三个归属截然不同的部分。**策略**归 provider：`RetryPolicyConfig` 的 schema 和解析器住在核心词汇包 `dsh-llm`（`retry-policy.ts`），每个适配器插件在自己的配置里嵌入 `RetryPolicySchema`，注册路由时被 `LlmRuntime` 捕获成不可变的 `ResolvedRetryPolicy`。默认值一目了然：

```ts
// packages/llm/llm/src/retry-policy.ts
const DEFAULT_MAX_RETRIES = 2
const DEFAULT_INITIAL_DELAY_MS = 500
const DEFAULT_MAX_DELAY_MS = 10_000
const DEFAULT_JITTER_RATIO = 0.1
const DEFAULT_RETRYABLE_CODES = Object.freeze([
  EMPTY_RESPONSE_CODE, 'RATE_LIMIT', 'SERVER', 'TIMEOUT', 'TRANSPORT',
])
```

`mode: 'normal'` 是有限次数 + 可重试码清单；`mode: 'always'` 无限重试、不看码（离线网关场景）。注意可重试码全是第 27、28 章铺好的 provider 中立 `code`——策略从不解析报错文案。

**关闭失败的一步**归 agent loop。回看 `agent.ts` 的 `step()`：流被完整消费（错误已被 `LlmRuntime` 归一化成终端 finish chunk 落进 `assistant/chunk` 日志）之后，loop 检查 finish：

```ts
// packages/core/agent-loop/src/agent.ts
      const finish = assembler.finish
      if (finish.kind === 'error' || finish.kind === 'aborted') {
        const action = await this.dispatch.waterfall(
          'agent/request-error', {
            turn, step,
            provider: request.provider,
            failure: finish.failure,
            retryPolicy: preparedCall?.retryPolicy,
            signal,
          },
          () => Promise.resolve<RequestErrorAction>(undefined),
        )
        signal.throwIfAborted()
        if (action?.kind !== 'retry') {
          throw new LlmError(finish.failure.message, finish.failure.code, finish.failure)
        }
        continue
      }
```

这就是「为什么不包 `llm/stream`」的答案：重试的正确位置在**一次尝试完整结束、失败已经落盘之后**。`agent/request-error` 是个 waterfall，payload 里带着失败 facts 和 `preparedCall` 捕获的 policy（上一章讲过，policy 随 registration 冻结，路由中途被替换也影响不到在途失败的恢复策略）。监听者返回 `{ kind: 'retry' }`，loop 的 `while (true)` 就 `continue` 重建请求再走一轮；没人认领，失败就成为 turn error。每次重试是一个全新的、完整记录的尝试——没有流中间的偷偷重来。

**执行器**才是 `llm-retry` 插件。它薄得惊人（226 行），而且配置为空——空得有态度：

```ts
// packages/llm/llm-retry/src/index.ts
function validateConfig(config: Config): void {
  const [key] = Object.keys(config)
  if (key === undefined) return
  if (key === 'retryPolicy') {
    throw new Error('llm-retry: retryPolicy belongs under each provider configuration')
  }
  throw new Error(`llm-retry: unknown key "${key}"`)
}
```

有人想在执行器上配策略？专门写一条报错把他指回 provider 配置。策略与执行的分界线被防守到了错误信息的措辞层面。

### recover()：从日志里数重试次数

执行器的核心是挂在 `agent/request-error` 上的 `recover()`。最值得看的是它如何知道「这是第几次重试」——不是内存计数器，是查日志：

```ts
// packages/llm/llm-retry/src/index.ts
    const policyKey = retryPolicyKey(policy)
    const priorPolicyRetry = agent.session.events.findLast((event): event is SessionEvent<'llm/retry'> =>
      event.type === 'llm/retry'
      && event.data.turn === turn
      && event.data.step === step
      && event.data.provider === provider
      && event.data.policyKey === policyKey,
    )
    const previousRetry = priorPolicyRetry?.data.retry ?? 0
    if (policy.mode === 'normal' && previousRetry >= policy.maxRetries) return next()
    const retry = previousRetry + 1
    const retryId = priorPolicyRetry?.data.retryId ?? RetryId(randomUUID())
```

计数键是 `(turn, step, provider, policyKey)` 四元组，`policyKey` 是策略参数的 JSON 序列化——中途改了 policy 或换了 provider，就是一条新的重试链，旧链的次数不会算到新策略头上。等待前后各落一条持久事件：

```ts
// packages/llm/llm-retry/src/index.ts (backoff)
    agent.session.append('llm/retry', eventData)          // 调度：含 failure、delayMs、retry/maxRetries
    if (!await cancellableDelay(delayMs, fusedSignal)) return
    agent.session.append('llm/retry-started', { retryId, turn, step, retry })
    return { kind: 'retry' }
```

「Each scheduled retry is durable before its cancellable wait」（模块头注释）——UI 能实时显示「正在第 2 次重试，还剩 3 秒」，进程崩溃后日志里也留着完整的失败与重试轨迹。延迟计算尊重 provider：`failure.providerRetryAfterMs`（来自 Retry-After 头，第 28 章的 `providerRetryAfterMs()` 解析）在不超过 `maxDelayMs` 时直接采用；超过时 normal 模式**放弃**（provider 要求等的时间超出策略容忍度，重试无意义）、always 模式退回本地指数退避。`always` 模式还有一层礼让：先 `await next()` 让更专业的下游恢复插件（比如换 key、修配置的监听者）先做修复，下游给出 `retry` 就采纳，下游抛错只记 warn 不拦路。收尾同样严谨：插件卸载时 `lifetime.abort` + `Promise.allSettled` 排干所有在途等待，waterfall 可能捕获的陈旧回调被 `lifetime.signal.aborted` 挡在门外。

配套的 `llm-retry-invariant` 伴生插件（`invariant.ts`，174 行，比主逻辑还认真）在开发态校验每条重试事件：必须落在 open turn/open step 内、provider 必须与 `request/header` 推导出的在途路由一致、`retry` 必须严格等于同链前值 +1、`retryId` 不得跨链复用、`llm/retry-started` 必须一一配对且不重复。重试链的形状本身成了被机器守护的日志不变量（[第 39 章](../part10/39-invariants-and-defensive-patterns.md)的模式）。

```mermaid
sequenceDiagram
    participant Loop as agent loop step()
    participant LLM as ctx.llm.stream
    participant Retry as llm-retry
    participant Log as session log
    Loop->>LLM: preparedCall.stream(request)
    LLM-->>Log: assistant/chunk × N（含 finish error）
    Loop->>Retry: waterfall agent/request-error {failure, retryPolicy}
    Retry->>Log: findLast llm/retry(turn,step,provider,policyKey)
    Retry->>Log: append llm/retry（先落盘）
    Retry->>Retry: cancellableDelay(delayMs)
    Retry->>Log: append llm/retry-started
    Retry-->>Loop: { kind: 'retry' }
    Loop->>LLM: 重建请求，新的一次完整尝试
```

### token-meter：锚定 provider、微分用启发式

`token-meter` 是一个 Cordis service（`ctx.tokenMeter`），同样不碰请求路径：它按需对每个 session 的事件日志做增量 fold（`WeakMap<Session, ReplayState>`，消费到哪条 seq 记在 `consumedEvents`），回答「下一个请求会有多大压力」。它的输入全部来自前两章铺好的日志材料：`request/header`（系统提示与工具 schema，[第 14 章](../part4/14-derive-messages.md)）、surface 事件（当前模型可见的消息面）、以及 `assistant/message` 携带的 provider `usage`。

核心难题是：provider 报的 usage 是唯一的真值，但它只描述**上一次**请求；此后 surface 又变了（新消息、compaction 缩了一段），而这些变化只能用启发式定价。`measure()` 的解法是「锚点 + 有符号增量」：

```ts
// packages/llm/token-meter/src/index.ts (measure)
    if (anchor !== undefined && optionalHeaderEquals(anchor.header, header)) {
      baseline = anchor.baseline
      surfaceDeltaTokens = state.surfaceTokens - anchor.surfaceTokens
    } else if (header === undefined && state.surfaceTokens === 0) {
      baseline = { kind: 'none', tokens: 0 }
      surfaceDeltaTokens = 0
    } else {
      baseline = { kind: 'estimated', tokens: estimateHeader(header) + state.surfaceTokens }
      surfaceDeltaTokens = 0
    }
    return deepFreeze(structuredClone({
      logRevision: state.consumedEvents,
      baseline, surfaceDeltaTokens,
      totalTokens: Math.max(0, baseline.tokens + surfaceDeltaTokens),
      surfaceTokens: state.surfaceTokens,
      nodes: state.surface,
    }))
```

最近一次成功调用的 usage 成为锚点（`baseline.kind === 'usage'`），之后 surface 的涨跌只按启发式**微分**叠加——绝对值来自 provider，误差只存在于增量里。锚点复用有两个先决条件：请求 envelope（canonical header）必须一致（system/tools 变了锚点作废），且 provider 总数不低于同时刻的全量启发式定价——fold 里那句注释点破原因：「Signed heuristic deltas remain conservative only from an anchor that is at least as large as the matching full heuristic price」。启发式本身故意粗糙且固定：`CHARS_PER_TOKEN = 4`，每块 4 token 结构开销（`estimate.ts`），服务名言明「the fixed estimator has no settings」——一个不许调参的估算器才能保证所有界面对同一内容报同一个数。

锚点构建还有个考究的细节：`assistant/message` 的 usage 描述的是 provider 在**流式输出时**看到的内容，而落盘消息可能已被后处理。于是 `_estimateProviderAssistant` 顺着上一章讲的 `sourceEventSeqs` 找回原始 `assistant/chunk` 事件，用 `BlockAssembler` 重新拼出 provider 原始输出来定价——第 27 章「日志存 chunk，消息存引用」的设计在这里兑现了第二次价值。

### 计量数据流向哪里

验证结论：token-meter **不追加任何 session 事件**，也不进 telemetry。usage 的持久化早在 loop 落 `assistant/chunk`（usage chunk）和 `assistant/message`（`usage` 字段）时完成了；token-meter 纯粹是读者。它的输出有两条通路：`measure()` 的即时快照（compaction 决策、状态栏），以及注册到 `dsh-session-projection` 通用投影注册表的三个纯 fold——`tokenUsage`（全会话累计的四个不相交桶：uncachedInput/output/cacheRead/cacheWrite，同一 step 重复上报做替换不做累加）、`contextPressure`（最新请求的 prompt 压力 + surface 微分出的 `projectedTokens`，O(1) 状态）、`contextBreakdown`（system/tools/messages 构成比）。投影注册本身是可选注入：

```ts
// packages/llm/token-meter/src/index.ts
    ctx.inject(['sessionProjections'], (projectionCtx) => {
      projectionCtx.sessionProjections.register(tokenUsageProjectionDefinition)
      projectionCtx.sessionProjections.register(contextPressureProjectionDefinition)
      projectionCtx.sessionProjections.register(contextBreakdownProjectionDefinition)
    })
```

没挂投影注册表的组合里，meter 保持独立可用。`ContextPressureProjection` 的文档注释还坦白了一个刻意的不一致：`contextWindow` 和 `pressureTokens` 是两个独立 last-wins 槽位，换模型的瞬间可能新容量配旧压力——「这是给人看的参考值，不是计费或门控输入」，与其为原子性把状态机复杂化，不如把非原子性写进文档。至于钱：全包上下没有一处价格表或币种字段，成本核算被留在 harness 之外——计量层输出不相交的 token 桶，正是为了让外部按各家价目表自行乘算。

## 设计亮点

> 💎 **设计亮点：重试计数器长在日志里。** `recover()` 用 `findLast` 从 session events 里推导「第几次、哪条链」，而不是插件内存里的 `Map<stepKey, count>`。普通写法在进程重启、插件 HMR、多实例并发时全部失忆或错乱；这里状态与事实同体——日志既是给人看的记录，也是算法的工作内存，且 crash 后自动恢复到正确的链位置。`policyKey` 入键更是神来之笔：策略变更自动开新链，语义上「换了规则就重新数」。

> 💎 **设计亮点：重试挂在失败已落盘之后，而不是流的中间。** 把执行器放在 `agent/request-error` 而非 `llm/stream` waterfall，意味着每次尝试都完整走完「chunk 落盘 → finish 归一化 → 失败关步」，重试是日志里两条一等公民事件，不是传输层里的一次静默循环。配合适配器侧 `maxRetries: 0`（第 28 章），「可见尝试」的账目在系统里只有一本。

> 💎 **设计亮点：provider 真值锚定 + 启发式微分。** token-meter 不在「全程 tokenizer 精算」（贵且口径对不上）和「全程估算」（误差无界）之间二选一，而是让 provider usage 提供绝对值、启发式只定价增量，并用「锚点不得低于全量启发式」的保守条件防止微分基线偏小。误差被结构性地限制在「上次请求以来的变化量」里。

> 💎 **设计亮点：用错误信息守卫架构分界。** `llm-retry` 的空配置校验对 `retryPolicy` 这个键单独回复「它属于 provider 配置」；`invariant.ts` 用 174 行校验重试事件链的每个字段。归属规则不只写在文档里，还写进了违反它时你会看到的那行报错。

## 小结与延伸

重试与计量共享一个世界观：请求路径只负责把事实忠实落盘（chunk、finish、usage、header），策略性的决定（要不要重试、还有多少余量）全部是日志的派生计算。policy 随 provider 注册冻结、executor 从日志数链、锚点从 usage 起算、投影按 seq 折叠——模型层由此获得一种少见的性质：任何时刻杀掉进程，重启后每个组件都能从日志推回自己该在的状态。这与[第 13 章](../part4/13-session-event-log.md)的事件溯源承诺首尾呼应，也是第七部分的收官注脚。

延伸阅读：

- `packages/llm/llm/src/retry-policy.ts` — 策略 schema、默认值与不可变解析
- `packages/llm/llm-retry/src/index.ts` / `types.ts` / `invariant.ts` — 执行器、持久事件声明与链不变量
- `packages/core/agent-loop/src/agent.ts`（`step()`）— `agent/request-error` waterfall 的发起点
- `packages/llm/token-meter/src/index.ts` / `estimate.ts` / `usage-projection.ts` / `projection.ts` — fold、启发式与三个投影
- `docs/subsystems/token-meter.md` — `TokenMeasurement` 语义的权威描述
