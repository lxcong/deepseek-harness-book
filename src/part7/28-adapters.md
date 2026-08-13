# 适配器接缝：llm-deepseek 与 llm-pi-ai

上一章的词汇表只是纸面契约，这一章看它被两次独立兑现：`llm-deepseek` 用裸 `fetch` + SSE 直连 DeepSeek 的 chat-completions 端点，`llm-pi-ai` 包装第三方库 `@earendil-works/pi-ai` 一口气服务多家 provider。两条实现路线截然不同，却对着同一份 `StreamChunk` 契约收敛——仓库文档里那句「the contract two implementations verified」就是这个意思。读完你会明白「加一个 provider 不动核心」在源码层面靠哪几个机关成立。

## 问题背景

朴素的多 provider 支持长这样：核心里一个 `switch (provider)`，每接一家加一个分支。三个月后的样子可以预言——核心包 import 了五家 SDK；某家 SDK 内置的自动重试和你自己的重试层叠加，一次失败悄悄打了六次请求；用户改了 API key 要重启进程才生效；两家对 tool call arguments 一个给字符串一个给对象，拼装代码里全是特判。

更隐蔽的坑在配置与凭证的时序上：请求进行到一半时配置变了，endpoint 用了新的、key 还是旧的（或反过来），产生一类几乎无法复现的 401。DeepSeek Harness 的两个适配器对这些坑各给出了工程化答案，而且答案惊人地一致：**每次操作捕获一份不可变快照**。

## 源码剖析

### 注册方式：同一个接缝，两种插件形态

两个插件都是标准 Cordis 插件（`name` + `inject: ['llm']` + `apply`），都通过 `ctx.llm.registerAdapter(providers, adapter)` 注册（[第 22 章](../part6/22-seam-triangle.md)的 provider 角色）。差异在路由的形状：

`llm-deepseek` 拥有一条写死的路由 `deepseek-official`，注册一次，之后只在「注册时被捕获的事实」（retry policy）变化时原地换血：

```ts
// packages/llm/llm-deepseek/src/index.ts
  const adapter = new DeepSeekAdapter({ options, resolveApiKey, resolveUserId })
  ctx.llm.registerConfigurableProviders([
    { provider: PROVIDER, displayName: 'DeepSeek', settingsNs: NS, settingsPath: [] },
  ])
  const registration = ctx.llm.registerAdapter([PROVIDER], adapter)
  let registeredPolicy = options().retryPolicy
  const ensureRegistrationFacts = (): void => {
    const policy = options().retryPolicy
    if (deepEqualJson(policy, registeredPolicy)) return
    // The registry captures the retry policy at registration, so it is the one
    // fact per-request resolution cannot refresh. `replace` re-reads it in one
    // synchronous registry section: disposing and re-registering instead would
    // publish an empty route set between the two...
    registration.replace([PROVIDER])
    registeredPolicy = policy
  }
```

`llm-pi-ai` 则拥有一**组**由配置决定的路由（`openai`、`anthropic`、自建网关……），支持「裸挂载」：零路由启动，等 settings 提供 profile 才注册，settings 清空则撤销。它把「哪些事实注册时被捕获」显式建模成一个可比较的值：

```ts
// packages/llm/llm-pi-ai/src/index.ts
/**
 * The registry captures these per route; a change here must re-register.
 * Sorted by provider so a settings document that merely reorders its keys is
 * not mistaken for a route change.
 */
function registrationFacts(profiles) {
  return [...profiles.entries()]
    .map(([provider, profile]) => ({
      provider,
      displayName: profile.displayName,
      retryPolicy: profile.retryPolicy,
    }))
    .sort((left, right) => left.provider.localeCompare(right.provider))
}
// ...
  const ensureRegistrationFacts = (): void => {
    const facts = registrationFacts(profiles())
    if (deepEqualJson(facts, registeredFacts)) return
    const routes = [...profiles().keys()]
    if (registration === undefined) {
      if (routes.length === 0) { registeredFacts = facts; return }
      registration = ctx.llm.registerAdapter(routes, adapter)
    } else {
      registration.replace(routes)
    }
    registeredFacts = facts
  }
```

两边都靠上一章讲的 `AdapterRegistrationHandle.replace()`：候选集先整体校验、再在一个同步区间内换掉，任何观察者都看不到「路由消失又出现」的空窗。除了适配器注册，`llm-pi-ai` 还注册 configurable-provider 目录（让配置界面能展示「可配置但尚未激活」的 provider）和 model discovery（替编辑中的草稿探测端点），三者共同构成设置页所需的全部能力——同样不动核心一行。

### 配置与凭证：thunk + 快照

两个适配器的构造函数都不接受静态配置，而是接受**解析钩子**。DeepSeek 侧：

```ts
// packages/llm/llm-deepseek/src/adapter.ts
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // One resolution per stream call: connection facts and the credential
    // freeze here and hold for this whole request...
    // The key resolves *from this snapshot*, so an endpoint and the secret
    // sent to it can never come from different configuration generations.
    const connection = this.config.options()
    const apiKey = await this.config.resolveApiKey(connection)
```

`options()` 每次调用重新解析 settings（带「上一份好配置」的降级：坏快照只记一条 error，继续用 lastGood），`resolveApiKey(connection)` 的签名强制 key 从**传入的这份快照**里解析——endpoint 和 secret 在类型签名层面就不可能来自两代配置。改 baseURL、改 key、改 catalog，下一个请求即刻生效，无须重启，也不打断在途请求。

pi-ai 侧的快照更重，因为库的 `Models.streamSimple()` 是惰性的——首次消费流时才解析 provider，那已经在 credential await 之后。模块头注释把因果讲得很透：如果原地改集合，一个请求可能「在旧配置下开始、在新配置下结束」。所以：

```ts
// packages/llm/llm-pi-ai/src/adapter.ts
  private current(): PiAiSnapshot {
    const profiles = this.config.profiles()
    if (this.snapshot?.profiles === profiles) return this.snapshot
    const models: MutableModels = createModels()
    for (const profile of profiles.values()) models.setProvider(profile.piProvider)
    this.snapshot = { profiles, models }
    return this.snapshot
  }
```

配置未变时按 profiles 的**对象恒等**复用快照；变了就整个重建集合，在途请求继续持有旧快照直到结束。这正是 `llm.prepareCall()` 那句「per-step call freeze 一直冻到底」的适配器侧下半场。凭证同样不进集合：harness 自己解析后作为请求级 `apiKey` 传给 pi-ai（其最高优先级覆盖），而且 profile 一旦**写了** `apiKeyEnv`，解析不到就抛 `MISSING_CREDENTIAL`——绝不把 `undefined` 递给库，否则库的环境变量自动发现可能捡到别家租户的 `OPENAI_API_KEY` 去计费。

### 请求组装：手写 wire vs 库的公共词汇

DeepSeek 适配器手工序列化（`serialize.ts`），几处注释暴露了真实世界的血泪：

```ts
// packages/llm/llm-deepseek/src/serialize.ts
  return {
    role: 'assistant',
    // Text-less turns send "" — NEVER null. ... Reasoning-ONLY turns (the model
    // can answer entirely in the reasoning channel...): the live API rejects
    // null-content/no-tool_calls assistant messages with a 400..., and since
    // the message sits durably in the session log, a null here bricks every
    // later turn of that session.
    content: text,
    // Official passback rule (guides/thinking_mode.mdx): reasoning_content
    // must return on tool-call turns; it is ignored on plain turns, so we
    // drop it there to save tokens.
    ...toolCalls.length > 0 && reasoning.length > 0 ? { reasoning_content: reasoning } : {},
    ...toolCalls.length > 0 ? { tool_calls: toolCalls } : {},
  }
```

注意那句「a null here bricks every later turn of that session」——因为 assistant 消息持久躺在 session log 里、每一步都会重放给 provider，一个序列化错误不是坏一次请求，是坏掉**整个会话的余生**。词汇表侧的其他翻译：harness 里 tool result 挂在 user 消息里，wire 上摊平成独立的 `{role:'tool'}` 消息；空 tool 输出补 `'(no output)'`；核心 image 块显式抛 `UNSUPPORTED_CONTENT` 而不是被 text-flatten 静默吃掉。

pi-ai 适配器不用碰 wire 格式（库管），它的组装工作是把 harness 词汇映射到库的公共选项（`toPiContext`、`profileOptions`），其中一行是全篇最重要的边界声明：

```ts
// packages/llm/llm-pi-ai/src/adapter.ts (profileOptions)
    // The agent recovery layer owns visible attempts; one adapter call is one SDK attempt.
    maxRetries: 0,
```

契约里「One adapter call is one provider attempt」的落点就在这——库的重试被显式关死，可见的重试全部交给第 29 章的 `llm-retry`，每次重试都是日志里一条持久事件，不存在暗地里的第六次请求。不支持的字段也走显式拒绝：`options.stop` 直接抛 `UNSUPPORTED_OPTION`，绝不静默丢弃。

### 流解析与 tool call：两种脏活，一份契约

DeepSeek 侧管道是 `fetch body → parseSse（eventsource-parser 负责 framing）→ translate`。`translate.ts` 的关键决策是**把 block-end、usage、finish 全部推迟到 `[DONE]` 哨兵**：

```ts
// packages/llm/llm-deepseek/src/translate.ts
    if (payload === DONE) {
      for (const block of order) {
        yield { type: 'block-end', index: block.index, block: closeBlock(block) }
      }
      if (pendingUsage) yield { type: 'usage', usage: pendingUsage }
      const reason = pendingFinish ?? { kind: 'stop' as const }
      yield {
        type: 'finish',
        reason: reason.kind === 'stop' && order.length === 0
          ? { kind: 'error',
              failure: { message: 'model returned a completed response with no content', code: EMPTY_RESPONSE_CODE } }
          : reason,
      }
      return
    }
```

为什么推迟？因为 provider 可能在 finish chunk 之后再补一个 usage-only chunk——如果 finish 到手就 yield，「usage 在 finish 前、finish 后无 chunk」的契约就破了。缓冲到 `[DONE]` 一次性按序 flush，两种 wire 形态都被同一段代码满足。这里还能看到两条契约条款的出生地：无内容的 `stop` 映射为 `EMPTY_RESPONSE` 错误（可重试，而不是一条空消息假装成功）；`mapUsage` 把 DeepSeek 「prompt_tokens 含缓存命中」的口径减回去，换算成词汇表的**不相交**计数。tool call 方面 OpenAI 风格天然是原始 JSON 字符串分片，`argumentsDelta` 直接透传即可。

pi-ai 侧（`stream.ts`）的事件已经是结构化的 `text_start/text_delta/.../done/error`，翻译几乎是逐条映射，但有两处方向相反的脏活。其一，pi-ai 交回的是**解析好的对象**，harness 词汇要求原始字符串，于是 `toolcall_end` 处 `arguments: JSON.stringify(event.toolCall.arguments)` 重新序列化。其二，pi-ai 从不 mid-stream 抛错，失败以终端 `error` 事件到达——恰好落进契约的第二条错误出口：

```ts
// packages/llm/llm-pi-ai/src/stream.ts
      case 'error':
        // In-stream error delivery (pi-ai's style) → error finish chunk
        // (the harness's other sanctioned error path besides throwing).
        yield { type: 'usage', usage: mapUsage(event.error.usage) }
        yield { type: 'finish', reason: mapStopReason(event.error, contextWindow) }
        return
```

代价是错误分类只能对库压平后的 message 文本做模式匹配（`classifyPiAiError` 上方一大段 `XXX(pi-ai upstream)` 注释，抱怨 undici 的 `cause` 链被上游丢弃）——直连实现里从 HTTP status 精确映射 `httpErrorCode()` 的活，在包库实现里退化成正则。两个适配器还各自把 provider 花样百出的「上下文超限」报错收敛到同一个 `CONTEXT_WINDOW_EXCEEDED` 码、用同一个 `idleWatchdog`（默认五分钟，仅在 `next()` 悬挂期间计时）把 provider 停摆映射为 `TIMEOUT`，且都区分「watchdog 到期」与「caller 主动 abort」。

### 扩展路径：cookbook 的最小形状

`docs/cookbook/adding-an-llm-adapter.md` 给出的全部核心代码只有：

```ts
class MyAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> { /* ... */ }
}
export const name = 'llm-myprovider'
export const inject = ['llm']
export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(/* ... */))
}
```

「加 provider 不动核心」成立的机关，拆开是四个：**路由即字符串键**——`GenerateOptions.provider` 选适配器，model id 原样透传，目录是 advisory 而非白名单，动态 catalog 的新模型无须任何注册就能用；**注册是 effect**——挂在插件 fiber 上，卸载/HMR 自动清路由（[第 6 章](../part2/06-reversible-effects.md)）；**能力是查询不是配置**——context window、reasoning efforts、默认 maxTokens 都由 `resolveModel()` 按 exact-model 回答，核心按需拉取；**策略随注册走**——retry policy 由 provider 声明、注册时捕获，执行器（llm-retry）完全不认识任何具体 provider。cookbook 同时列出义务清单：usage/finish 顺序、raw JSON arguments、两条错误出口、honor `options.signal`、不支持的字段抛 `UNSUPPORTED`——正是上一章契约的作者视角复述。

## 设计亮点

> 💎 **设计亮点：把「配置一致性」编码进函数签名。** `resolveApiKey: (connection: DeepSeekConnectionOptions) => Promise<string>` 强制凭证从调用方传入的快照解析；pi-ai 的 `PiAiSnapshot` 让 profiles 与 Models 集合同生共死。普通写法是各处各自读一遍「当前配置」，endpoint 与 key 的代际错配只能靠纪律防守；这里靠数据流形状防守——想写出错配代码，先得绕过类型。

> 💎 **设计亮点：`maxRetries: 0` 是一条架构声明。** 一行配置把「重试」从传输层没收，上交给 agent 恢复层。换普通写法（放任 SDK 默认重试），日志里一次失败背后可能是 N 次隐形请求，llm-retry 的持久事件、退避计算、maxRetries 语义全部失真。一个适配器调用 = 一次 provider 尝试，是让第 29 章一切成立的地基。

> 💎 **设计亮点：缓冲到 `[DONE]` 再 flush，用一段代码满足两种 wire 形态。** finish 附带 usage、finish 后补 usage-only chunk，这两种 provider 行为若逐 chunk 直译就需要状态机特判，还容易破坏「finish 之后无 chunk」的契约。把终结性 chunk 全部延迟到哨兵处按固定顺序输出，协议约束从「小心维护」变成「结构上不可能违反」。

> 💎 **设计亮点：目录是 advisory，注册才是 authority。** `listModels()` 喂选择器，但请求路由从不校验 catalog 成员资格——描述一个模型失败不该让整个 provider 从 picker 里消失（`describableReasoningLevel` 的注释把这个取舍写得明明白白），而请求路径该拒绝的照样拒绝。查询与执行的失败域被有意切开。

## 小结与延伸

两个适配器是同一份契约的两次独立证明：直连实现展示了 wire 层每个鲁棒性细节的来历（null content 砖掉会话、`[DONE]` 缓冲、disjoint usage 换算），包库实现展示了如何驯服一个有自己想法的 SDK（惰性求值逼出不可变快照、关掉内置重试、重新字符串化 arguments）。它们共同验证的机关——字符串路由、effect 注册、能力查询、策略随注册——就是「加 provider 不动核心」的全部答案。下一章顺着 provider 声明的 retry policy，看失败之后发生的事。

延伸阅读：

- `packages/llm/llm-deepseek/src/` — `serialize.ts` / `sse.ts` / `translate.ts` / `adapter.ts` / `index.ts`，cookbook 钦定的分层参考
- `packages/llm/llm-pi-ai/src/` — `adapter.ts` / `stream.ts` / `index.ts`；`config.ts` 与 `catalog.ts` 是 profile 解析的另一半
- `docs/cookbook/adding-an-llm-adapter.md` — 扩展路径与义务清单
- `packages/llm/llm-deepseek/tests/mock-server.ts` — 两个适配器如何被 wire 级测试钉住（含 attribution header 证明）
