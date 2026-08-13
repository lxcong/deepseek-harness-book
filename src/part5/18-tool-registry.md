# 第 18 章 工具注册表与作用域注册

工具是 Agent Harness 的手脚，而工具注册表决定了「模型看到什么、能调什么、谁说了算」。这一章读 `packages/core/tools` 的注册侧：`ctx.tools` 的注册 API、工具 schema 的类型化方案（顺带回答一个悬案：它用没用仓库里那套 typert？）、per-agent 的 scoped registration，以及工具 schema 如何汇入每次 prompt assembly。执行流水线留给[第 19 章](19-tool-pipeline.md)。

## 问题背景

朴素做法是一张全局 `Map<string, ToolFn>`：插件往里塞，循环从里取，把所有 value 序列化成 JSON Schema 发给模型。这个做法会在四个地方翻车：

1. **类型断裂**。JSON Schema 是运行时数据，TypeScript 看不见它。工具作者在 `parameters` 里声明了 `path: string`，在 `execute(args)` 里拿到的却是 `unknown`，两边全靠人肉对齐，schema 改了实现不改，模型传来的参数静默错位。
2. **多 agent 冲突**。一个进程里跑主 agent 和若干 subagent，各自需要不同的工具面。全局 Map 只有一个答案：要么所有 agent 看到同一套工具，要么在调用点写 if-else。
3. **注册与展示脱节**。注册表和 system prompt 是两个模块，工具注册了但 schema 没进 prompt（模型不知道有这个工具），或者 prompt 里有但注册表里已经卸载（模型调一个不存在的名字），两种漂移都要靠纪律防守。
4. **热重载泄漏**。插件卸载时忘了从 Map 里删自己的工具，下一次加载就是重复注册。

DeepSeek Harness 的答案是：注册表本身是一个 Cordis Service（`ctx.tools`），注册返回 disposer、按 scope 分层、schema 用自研 DSL 做编译期推断 + 运行期校验，并且注册表**主动把自己接到 prompt assembly 上**。

## 源码剖析

### ctx.tools：declaration merging 挂载的服务

`ToolRuntime` 通过 declaration merging 把自己合并进 Cordis 的 `Context` 接口，任何插件拿到 `ctx` 就能类型安全地访问 `ctx.tools`（这套机制见[第 4 章](../part2/04-context-and-service.md)）：

```ts
// packages/core/tools/src/index.ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    tools: ToolRuntime
  }
  interface Events {
    'tools/pre-execute'(/* ... */): Promise<PreToolDecision>
    // ... tools/execute、tools/post-execute、tools/result、tools/change
  }
}
```

注册 API 是 `register(definition)`。注意它做的不只是插表：

```ts
// packages/core/tools/src/index.ts
register(definition: ToolDefinition): () => void {
  const name = definition.name
  const output = (definition as Partial<ToolDefinition>).output
  if (output === undefined || typeof output !== 'object'
    || typeof output.render !== 'function' /* ... */) {
    throw new TypeError(`tool "${name}" must declare output { schema, render, presentationMeta? }`)
  }
  assertSupportedJsonSchema(output.schema)
  // ...
  if (name === RUN_CODE_NAME) {
    throw new Error(`tool name "${RUN_CODE_NAME}" is reserved for the Code Mode presentation transport ...`)
  }
  return this.layers.effect(
    this.ctx,
    layer => layer.tools.insert(name, definition),
    { label: 'tools.register()' },
  )
}
```

三个细节：其一，`output` 声明是**强制的**——每个工具必须声明规范输出的 JSON Schema 和一个纯函数 `render`，把结构化 value 投影成模型可见的 `ContentBlock[]`（这是第 19 章输出校验的前提）。其二，`run_code` 这个名字被无条件保留，因为任何 agent 随时可能给自己切到 Code Mode，一个「当前没被占用」的名字下一秒就可能变成碰撞。其三，返回值是 `this.layers.effect(...)` 的产物——精确的 disposer，插件卸载时 Cordis effect 机制自动回收注册（[第 6 章](../part2/06-reversible-effects.md)的可逆副作用在这里落地）。

### 工具 schema 的类型化：不是 typert，是自研 DSL

仓库里确实有一个 `packages/typert/`，但查证 `packages/core/tools/package.json` 会发现它的运行时依赖只有一个：

```json
"dependencies": {
  "@deepseek-ai/schemastery": "workspace:^"
}
```

typert 是构建期的类型模型生成器（把 TypeScript 类型树抽成 `FaceModel`，为 API catalog 和 client 契约生成 Zod schema），它服务于 host/client 边界，**与工具参数 schema 无关**。工具 schema 的类型化是 `packages/core/tools/src/schema.ts` 里一套自研的 value schema DSL；schemastery 只用于插件 config（`static Config`）的校验。

DSL 的作者面是一组带字面量约束的 spec 类型，配一个关键的推断类型 `InferValue`：

```ts
// packages/core/tools/src/schema.ts
export type ParameterSchemaSpec = {
  [key: string]: ParameterPropertySpec   // ValueSchemaSpec & { required?: true }
  [key: symbol]: never
}

/** Infer one node without recursively checking it against the full author union. */
type InferValueAt<S, Depth extends unknown[]> =
  Depth['length'] extends 16 ? JsonValue :
    S extends { type: 'string' } ? InferScalar<S, string> :
      S extends { type: 'number' | 'integer' } ? InferScalar<S, number> :
        // ... boolean / null / array / object / json / oneOf
        S extends { oneOf: readonly unknown[] }
          ? InferValueAt<S['oneOf'][number], NextInferenceDepth<Depth>>
          : never

export type InferValue<S> = InferValueAt<S, []>
export type InferArgs<S> = InferProperties<S, []>
```

`defineTool` 用 `const` 泛型把作者写的 schema 字面量原样保留进类型系统，再在运行时把同一份 spec 编译成 JSON Schema 并校验模型参数：

```ts
// packages/core/tools/src/schema.ts
export function defineTool<const S extends ParameterSchemaSpec, const O extends ValueSchemaSpec>(
  options: DefineToolOptions<S, O>,
): ToolDefinition {
  // ...
  const parameters = parameterSchemaSpecToJsonSchema(options.parameters)
  const validate = (args: unknown): string[] => validateJsonSchemaValue(parameters, args, '')
  const tool: ToolDefinition = {
    // ...
    async execute(args: unknown, exec: ToolRunContext): Promise<JsonValue> {
      const violations = validate(args)
      if (violations.length > 0) throw new ToolArgsError(violations)
      return userExecute(args as InferArgs<S>, exec) as Promise<JsonValue>
    },
  }
  // ...
}
```

于是同一份 `parameters` spec 有三个消费者：编译期给 `execute(args: InferArgs<S>)` 提供精确类型；运行期挡掉模型生成的坏参数（`ToolArgsError`，code `INVALID_ARGS`，会变成模型可见的错误结果）；投影期变成发给模型的 JSON Schema。三者同源，改一处三处同步。值得对照的是展示回调的处理——`presentCall`/`presentResult` 走的是**软校验**：校验失败不抛错，返回 `undefined` 落回通用卡片。注释讲得很直白：展示可能在回放旧日志时运行，日志里的参数可能来自旧版 schema，展示层永远不该 throw。

推断深度上限 16 层（`Depth['length'] extends 16 ? JsonValue`）是对 TypeScript 递归极限的防御：超深的容器嵌套优雅降级到 `JsonValue`，而不是让整个工程的类型检查爆栈。

### Scoped registration：per-agent 工具面

注册表内部不是一张 Map，而是一组按 `ScopeKey` 分层的 `ToolLayer`（scope 原语见[第 11 章](../part3/11-scope.md)）。通过普通 `ctx` 注册是全局工具，通过 `agent.ctx` 注册只属于那个 agent。解析可见性的核心是 `view()`：

```ts
// packages/core/tools/src/index.ts
private view(scope?: ScopeKey): ToolView {
  const layers = this.layers.chainLayers(scope)
  const own = this.layers.peek(scope)
  // Inherited surface, nearest ancestor last: a nearer scope's same-name
  // entry shadows a farther one, and the global layer is the farthest.
  const inherited = new Map<string, ToolDefinition>(this.layers.global.tools.entries())
  for (const layer of layers) {
    if (layer === own) continue
    for (const [name, definition] of layer.tools.entries()) inherited.set(name, definition)
  }
  const visible = new Map<string, ToolDefinition>()
  // ...
  for (const [name, definition] of inherited) {
    // Restrictions intersect across the whole chain
    if (layers.every(layer => layer.admits(name))) visible.set(name, definition)
  }
  // The scope's own registrations last, shadowing an inherited name and
  // outside the filter above.
  if (own !== undefined) {
    for (const [name, definition] of own.tools.entries()) visible.set(name, definition)
  }
  if (this.modeFor(scope) !== 'native') {
    visible.set(RUN_CODE_NAME, this.requireCodeTransport())
  }
  return { visible, knownNames, restrictableNames }
}
```

规则可以概括为三条：**继承**（agent 沿 scope 链继承全局与祖先层的工具，近者 shadow 远者）；**过滤**（`tools.restrict({ allow, deny })` 声明的限制在整条链上取交集，但只过滤*继承来的*名字，不碰本层自己的注册）；**遮蔽**（本层同名注册覆盖继承）。「限制不碰本层注册」这条豁免不是偷懒——`view()` 上方的长注释解释了它保护的场景：委托运行时会把子 agent 的汇报工具注册进子 agent 自己的层，父级给子 agent 的能力过滤绝不能把「答题通道」一起滤掉。注释还记录了一次真实的踩坑：早期把豁免集合理解成「全局层」，在工具从 host 组合迁到 agent plane 之后，子 agent 的 filter 静默失效——这段历史被写进了源码注释而不是仅存在于 postmortem。

`restrict()` 本身也体现了 fail-loud 的风格：空 filter 直接抛错（「an empty filter is almost always a materialized-empty-config bug」），未知名字抛错并列出全部已知名字，未 scoped 的 context 调用抛错并告诉你正确做法。

### 工具 schema 如何进入 prompt assembly

注册表不等别人来拉 schema，而是在构造时把自己注册为 system prompt 的 tool provider：

```ts
// packages/core/tools/src/index.ts
constructor(ctx: Context, config: Config = {}) {
  super(ctx, 'tools')
  // ...
  ctx.systemPrompt.tools(context => this.wireSchemas(context.scope))
  if (this.defaultMode !== 'native') {
    ctx.systemPrompt.section(this.collapseSection())
    ctx.systemPrompt.section(this.sdkSection())
  }
}

private wireSchemas(scope?: ScopeKey): ToolProviderResult {
  const view = this.view(scope)
  const mode = this.modeFor(scope)
  if (mode === 'native') {
    const schemas = [...view.visible.values()].map(definition => this.schemaOf(definition, false))
    return { schemas, knownNames: [...view.knownNames] }
  }
  // ... code / both 模式：只发 run_code，或两者都发
}
```

每次 assembly（每个模型 step 之前）都会以**当前调用 scope**重新求值：agent A 的 prompt 里是 A 的可见工具集，agent B 是 B 的，全部出自同一个 `view()`——展示、查找、执行三个入口共享一个可见性解析器，模型「看到的」和注册表「肯执行的」由构造保证一致。`knownNames` 与 `schemas` 分开返回也有讲究：前者是**限制前**的名字全集，供 `toolOrder` 配置校验用——一个名字被某个 scope 限制隐藏是合法的（该 scope 缺席即可），但配置里写了一个从未注册过的名字要在 assembly 时炸出来。这些如何被拼进最终 prompt，见[第 21 章](21-system-prompt.md)。

`schemaOf` 只投影 `name`/`description`/`parameters` 三个白名单字段——`execute`、`timeoutMs`、`presentCall` 这些执行与展示回调**从不**发往模型，模型面与实现面在投影处强制分离。

## 设计亮点

> 💎 **设计亮点：一份 schema spec，三个消费者。** 普通写法是 TypeScript interface 和 JSON Schema 各写一份，靠 code review 保持同步。`defineTool` 用 `const` 泛型 + 条件类型推断（`InferArgs`/`InferValue`），让编译期类型、运行期校验、模型面 JSON Schema 全部从同一个字面量 spec 派生，输出 schema（`output.schema`）也享受同样待遇。而且没有引入 zod 这种重依赖——16 层深度上限的自研推断对工具参数这个域足够，还避免了推断爆栈。

> 💎 **设计亮点：执行路径硬校验，展示路径软校验。** 同一个 `validate`，在 `execute` 里失败抛 `ToolArgsError`（模型必须知道自己传错了），在 `presentCall`/`presentResult` 里失败返回 `undefined` 落回通用卡片（回放旧日志时参数可能来自旧 schema，UI 不该炸）。一个校验器，两种失败语义，各自匹配调用场景的容错需求。

> 💎 **设计亮点：限制过滤继承面、豁免本层注册。** `tools.restrict()` 的语义边界精确到「not mine」而不是「非全局」。这个一词之差在工具从 host 层迁移到 agent plane 时决定了子 agent 的能力过滤是否静默失效，而源码注释把这次架构迁移引发的语义漂移完整记录在了豁免逻辑旁边——防御的不只是 bug，还有未来读者的误解。

> 💎 **设计亮点：注册表主动接入 prompt assembly。** 不是 prompt 模块 import 工具模块（耦合），也不是靠人肉在两处同步（漂移），而是 `ToolRuntime` 构造时调 `ctx.systemPrompt.tools(provider)` 把自己挂上去，provider 按 assembly 时刻的 scope 现算。「模型看到的工具」永远是「注册表此刻肯执行的工具」的投影，一致性由数据流方向保证。

## 小结与延伸

`ctx.tools` 是一个分层注册表：注册返回可逆 disposer，schema 用自研 DSL 做「一份 spec 三处消费」，per-agent 层通过继承 + 交集限制 + 遮蔽合成每个 agent 的工具面，再由同一个 `view()` 喂给 prompt assembly 与执行解析。typert 存在于仓库但与工具 schema 无关——它是构建期契约生成器。下一章进入这张注册表的另一半：调用如何穿过三段 waterfall 落地为持久化的 `tool/result`。

延伸阅读：

- `packages/core/tools/src/schema.ts` — DSL 全文，重点看 `runSchemaCompiler` 的非递归编译
- `packages/core/tools/src/json-schema.ts` — 被强制执行的 JSON Schema 子集
- `packages/core/tools/tests/scoped.spec.ts`、`tests/schema.spec.ts` — 行为契约
- `docs/subsystems/tools.md`、`docs/cookbook/adding-a-tool.md` — 官方地图
- [第 11 章](../part3/11-scope.md) — `ScopedLayers` 的通用机制
