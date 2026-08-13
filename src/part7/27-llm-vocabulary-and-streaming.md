# LLM 词汇表与流式协议

这一章读 `packages/llm/llm`——整个 Harness 的模型层地基。它做两件事：定义一套 provider 中立的 message/stream「词汇表」，以及提供一个可被 waterfall 拦截的流式调用服务 `LlmRuntime`。读懂这一章，你会明白为什么 agent loop、session log、UI、compaction 可以对「模型说了什么」达成共识，而不用知道背后是 DeepSeek 的 SSE 还是 pi-ai 库的事件流。这也是后两章（适配器、重试与计量）的公共前提。

## 问题背景

自己实现模型层时，最省事的做法是直接把某家 provider SDK 的类型透传出去：`ChatCompletionChunk` 一路流到 UI。坑会在三个时刻爆发：

第一，接第二家 provider 时。两家的流式协议对「一个 tool call 分几段传、arguments 是字符串还是对象、usage 什么时候出现」全都不同，于是每个消费方都长出 `if (provider === ...)` 分支。

第二，把对话写进持久化日志时。日志里存的是某家的 wire 格式，换 provider 后老会话无法重放，甚至无法渲染。

第三，处理流式重组时。每个想要「完整消息」的消费方（日志、UI、工具执行器）都各自把 delta 拼回块，三份拼装逻辑在边界条件上各错各的。

DeepSeek Harness 的答案是：把**词汇表**（类型与协议约定）收进一个几乎零依赖的包 `dsh-llm`，把**翻译**（wire ↔ 词汇表）全部推给适配器包，再用**一个**共享的 `BlockAssembler` 消灭重复拼装。

## 源码剖析

### 一套消息词汇：merge-extensible 的 interface map

会话就是 `Message[]`，一条消息就是一组带类型标签的 content block。块的并集不是写死的 union，而是从一个可 declaration-merging 的 interface map 推导出来：

```ts
// packages/llm/llm/src/types.ts
/**
 * Merge-extensible content blocks keyed by `type`. New core blocks must land
 * with adapter, UI, and compaction support.
 */
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}

export type ContentBlockType = keyof ContentBlockMap
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

这与[第 5 章](../part2/05-typed-events.md)事件系统的套路一致：插件可以 `declare module` 往 map 里加自己的块类型，消费方 `switch` 时对未知类型做 fall-through。同样的手法还用在 `FinishReasonMap`（finish 原因可扩展）和 `MessageSourceMap`（消息来源可扩展）上。

`Message` 本体只有四个字段：`id`（branded `MessageId`）、`role`、`content`、`source`。有意思的是 `source` 的设计——它拆成了两条独立的轴：

```ts
// packages/llm/llm/src/message.ts
export interface MessageSourceMap {
  user: { kind: 'user' }
  plugin: { kind: 'plugin'; plugin: string } & ContextFormed
  model: ModelMessageSource
  tool: ToolMessageSource
}
```

`kind` 回答「谁产生的」，可选的 `form`（`instructions` / `catalog` / `snapshot` / `notice` / `relay` / `recall`）回答「这是什么性质的内容」。源码注释特意强调这套词汇是 **SEMANTIC, never visual**——`form` 只说「这是一份目录」，怎么渲染（颜色、折叠、图标）是消费方的事。而且 `ContextFormed` 是按 `form` 判别的 union：选了 `notice` 就必须带 `summary`，选了 `snapshot` 就必须带 `sections`，类型系统保证「选了形式却缺渲染所需字段」这类错误不可能编译通过。

模型产生的 assistant 消息还要带上「出处」：

```ts
// packages/llm/llm/src/message.ts
export interface AssistantProvenance {
  provider: string
  model: string
  /**
   * Lossless-JSON adapter state needed to replay the provider response.
   * `LlmRuntime` exposes it to a target adapter only when that adapter instance
   * currently owns both this historical provider and the target provider.
   */
  replayState?: unknown
}
```

`replayState` 是适配器的私有重放状态（比如 provider 要求后续请求带回的 response id）。谁能读它的规则很讲究，后面会讲到。

### StreamChunk：一个刻意「封闭」的流式协议

与前面的 merge-extensible map 相反，流式 chunk 是一个**封闭的** discriminated union：

```ts
// packages/llm/llm/src/types.ts
export type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: CallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | {
    type: 'finish'
    reason: FinishReason
    /** Adapter-private lossless-JSON state for replaying a successful response. */
    replayState?: unknown
  }
```

一次响应可以交错多个块（文本、思考、多个 tool call），`index` 把 delta 归到各自的块上；`block-end` 直接携带**拼装完成的** `ContentBlock`。消费 chunk 的 `switch` 以 `assertNever` 收尾，所以给协议加一个变体会让每个消费方编译失败——协议演进是显式的、全量的。

协议还带一组适配器必须遵守的约定（`docs/subsystems/llm-streaming.md` 列了全表，且强调这是「两个真实实现共同验证过的契约」）：`usage` 必须在 `finish` 之前、`finish` 之后不许再有任何 chunk；tool call 的 `arguments` 全程保持**原始 JSON 字符串**；错误只有两条合法出口——从 `stream()` 抛 `LlmError`，或者以 `finish {kind:'error'|'aborted', failure}` 结束流。两条出口共享同一个可序列化的 `LlmFailure`（`message` + 稳定 `code` + 可选 `status`/`providerRetryAfterMs`/`requestId`），下游（重试策略、UI）只路由 `code`，永远不解析 provider 的报错文案。

### llm/stream waterfall 与错误归一化

`LlmRuntime.stream()` 不直接调适配器，而是把终端适配器包在一个 waterfall 里：

```ts
// packages/llm/llm/src/index.ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    /**
     * Waterfall around every streaming model call (retry, replay, routing).
     * ...
     * @mode waterfall
     */
    'llm/stream'(this: LlmRuntime, options: GenerateOptions, next: () => AsyncIterable<StreamChunk>): AsyncIterable<StreamChunk>
  }
}

  private streamWithRegistration(options, prepared?) {
    return this.ctx.waterfall(
      this, 'llm/stream', options,
      () => this.adapterStream(options, prepared),
    )
  }
```

任何插件都可以监听 `llm/stream`：调 `next()` 走到真正的适配器，或者自己 yield chunk 短路整个调用（mock、replay、路由都是这一个原语）。loop 组装的请求会被 `markAgentLoopRequest()` 打上进程内 WeakSet 标记并 deep-freeze——监听者可以读，改就会 throw，因为 loop 请求的内容是 session log 的纯函数（[第 15 章](../part4/15-model-visible-invariant.md)的不变量），改写它等于让日志失真。

终端边界 `adapterStream` 是本章最细腻的一段控制流：

```ts
// packages/llm/llm/src/index.ts
  private async * adapterStream(options, prepared?): AsyncGenerator<StreamChunk> {
    let iterator: AsyncIterator<StreamChunk>
    try {
      // ... 选 registration、resolve config、adapter.stream(...)
      iterator = stream[Symbol.asyncIterator]()
    } catch (error: unknown) {
      yield adapterFailureChunk(error, options.signal)
      return
    }
    let completed = false
    try {
      while (true) {
        let item: { done: true } | { done: false; value: StreamChunk }
        try {
          const next = await iterator.next()
          item = next.done ? { done: true } : { done: false, value: next.value }
        } catch (error: unknown) {
          completed = true
          yield adapterFailureChunk(error, options.signal)
          return
        }
        if (item.done) { completed = true; return }
        // End the adapter-owned try before yielding: consumer/middleware
        // failures resumed into this generator must remain thrown.
        yield item.value
      }
    } finally {
      if (!completed) { /* iterator.return() 清理 */ }
    }
  }
```

注意 `yield item.value` 被刻意放在 try 块**外面**。async generator 的 `yield` 是双向的：如果消费方在 `for await` 体里抛错，这个错误会「resume」进 generator 的 yield 点。要是 yield 在 try 里，消费方自己的 bug 就会被误判成适配器失败、变成一个 `finish error` chunk 静静躺进日志。现在的写法保证：**适配器的失败**归一化为终端 finish chunk（经 `normalizeLlmFailure` 提取可序列化 facts），**消费方与中间件的失败**保持 throw 语义。错误归属在类型层面就分了家。

### BlockAssembler 与 assistant/chunk 的衔接

流式协议和 session log 在 agent loop 的 `step()` 里合流（[第 9 章](../part3/09-agent-loop-deep-dive.md)）：

```ts
// packages/core/agent-loop/src/agent.ts
      const assembler = new BlockAssembler()
      const chunkSeqs: number[] = []
      const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
      for await (const chunk of stream) {
        signal.throwIfAborted()
        chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
        assembler.push(chunk)
      }
      // ...
      this.session.append('assistant/message',
        { turn, step, message, ...assembler.usage === undefined ? {} : { usage: assembler.usage } },
        { surfaceOp: 'append', sourceEventSeqs: chunkSeqs },
      )
```

每个原始 chunk 原样落一条 `assistant/chunk` 事件（重放保真），同一个 chunk 同时喂给 `BlockAssembler`；流结束后拼出的 assistant 消息作为 `assistant/message` 落盘，并通过 `sourceEventSeqs` **引用**它由哪些 chunk 事件拼成。第 29 章会看到 token-meter 正是顺着这些 seq 反向重放拼装的。

`BlockAssembler`（`packages/llm/llm/src/assembler.ts`）是全仓唯一的拼装实现，几个细节值得记：容忍没有 block-start/end 的 delta-only 协议（`ensure()` 按需开块）；`block-end` 之后再来的 delta 直接忽略——「first close wins」，让流式展示与最终拼装结果永远一致，且失控的适配器无法撑爆内存；`blocks()` 在 finish 为 `max-tokens` 时会**过滤掉 tool-call 块**，因为被截断的 tool call 参数不完整，执行它不安全。

### 为什么词汇表与适配器要分开

`dsh-llm` 里没有一行 HTTP 代码。`LlmAdapter` 抽象类只要求一个方法：

```ts
// packages/llm/llm/src/index.ts
export abstract class LlmAdapter {
  providerInfo(provider: string): LlmProviderInfo { /* 默认实现 */ }
  providerRetryPolicy(_provider: string): ResolvedRetryPolicy | undefined { return undefined }
  listModels(_provider: string): Promise<readonly LlmModelInfo[]> { /* 默认 [] */ }
  resolveModel(provider, model, _signal?): Promise<LlmResolvedModelInfo> { /* 默认恒等 */ }
  /** Stream one model call as raw chunks. The only required method. */
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>
}
```

分开的收益是三重的。其一，依赖方向：agent loop、session、tools、UI 全都只依赖词汇表包，适配器包依赖词汇表——加 provider 不需要任何核心包 rebuild（下一章展开）。其二，归属清晰：`ToolSchema` 声明在 `dsh-llm` 而不是 `dsh-tools`，因为它是 `GenerateOptions` 的一部分——「发给模型的东西」属于模型层词汇。其三，安全的跨 provider 历史：`replayState` 是适配器私有状态，`LlmRuntime.forAdapter()` 在派发前遍历消息，只有当历史消息的 provider 与目标 provider **当前注册在同一个适配器实例**上时才保留它，否则剥掉 `replayState` 只留 provider 中立的内容。换了适配器，历史照样可读，私有状态绝不会喂给不认识它的实现。

另外值得一提的是 `prepareCall()`：loop 每一步先 `prepareCall` 拿到一个一次性句柄，它把「exact-model 元数据解析、请求头落盘、真正派发」绑死在**同一个** adapter registration 上——热重载（HMR）换掉适配器时，不会出现「用 A 适配器的能力元数据、却派发到 B 适配器」的撕裂；句柄二次派发或配置漂移都会以 `INVALID_PREPARED_CALL` 拒绝。

## 设计亮点

> 💎 **设计亮点：开放的词汇、封闭的协议。** `ContentBlockMap` / `FinishReasonMap` / `MessageSourceMap` 用 declaration merging 保持开放（插件可加块类型、加消息来源），而 `StreamChunk` 是封闭 union + `assertNever`。普通写法要么全开放（协议变更悄无声息地漏掉消费方），要么全封闭（插件没有扩展点）。这里按「谁需要演进、演进代价由谁承担」分别选型：内容词汇的新成员必须「带着 adapter、UI、compaction 支持一起落地」（类型注释原文），流协议的新变体则让每个 `switch` 编译失败逼你处理。

> 💎 **设计亮点：一个 yield 位置的讲究。** `adapterStream` 把 `yield` 移出 try 块，用 async generator 的 resume 语义精确切分错误归属：适配器的 throw 变成可序列化的终端 finish chunk（进日志、进重试策略），消费方的 throw 保持异常传播（是 bug 就该炸）。换普通写法——整个循环包一个 try——消费方的 bug 会被吞成「模型调用失败」，排查时方向完全错误。

> 💎 **设计亮点：replayState 按「适配器实例身份」放行。** 判据不是 provider 名字符串相等，而是 `this.adapters.get(source.provider)?.adapter === adapter`——同一个实例、当下同时拥有历史路由和目标路由，才配读自己写下的私有状态。cookbook 里还特意警告「absent 时绝不能凭 provider/model 名推断可以 native replay」。字符串同名但实现已换（HMR、配置改动）的场景被这一个恒等比较全部挡住。

> 💎 **设计亮点：日志存 chunk，消息存引用。** loop 把原始 chunk 逐条落 `assistant/chunk`，再把拼好的 `assistant/message` 通过 `sourceEventSeqs` 指回去。普通做法只存最终消息——省空间，但「模型实际逐字说了什么」永久丢失，重放、计量、排障都失去原料。这里用 append-only 日志承担全部真相，拼装结果只是它的一个投影（[第 13 章](../part4/13-session-event-log.md)）。

## 小结与延伸

`dsh-llm` 用不到两千行定义了模型层的全部公共语言：merge-extensible 的消息词汇、封闭且带硬性契约的 `StreamChunk` 协议、归一化两条错误出口的 `LlmRuntime`、以及全仓唯一的 `BlockAssembler`。适配器只翻译，不定义；核心只消费词汇，不认识 wire。下一章看两个真实适配器如何各自兑现这份契约，第 29 章看重试与计量如何完全建立在本章的 `LlmFailure`、`usage` chunk 与 `assistant/chunk` 日志之上。

延伸阅读：

- `packages/llm/llm/src/types.ts` / `message.ts` — 词汇表全文
- `packages/llm/llm/src/index.ts` — `LlmRuntime`、waterfall、错误归一化
- `packages/llm/llm/src/assembler.ts` — 拼装器与容错规则
- `packages/llm/llm/src/call-config.ts` — `markAgentLoopRequest` 与跳过 AbortSignal 的 `deepFreeze`
- `docs/subsystems/llm-streaming.md` — 协议契约的权威清单（含生成式 Cordis API 目录）
