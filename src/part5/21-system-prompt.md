# 第 21 章 System Prompt 的组装

在「everything is a plugin」的架构里，system prompt 不能是一个手写的长字符串——工具指引来自工具插件、身份来自宿主、persona 来自部署或 preset、审批策略叙述来自审批插件，它们的生命周期各不相同，却要在每个模型 step 前拼成同一段文本。这一章读 `packages/core/system-prompt`：`ctx.systemPrompt` 的 section 注册机制、各插件如何贡献段落、拼装顺序与跨机器稳定性、工具 schema 如何汇入同一次 assembly，以及「prompt 也是插件贡献的」这个决定对可扩展性意味着什么。

## 问题背景

朴素做法是模板拼接：`SYSTEM_PROMPT = identity + persona + toolGuide + ...`，写在 agent loop 里。它的裂缝随插件化立刻显现：

- **贡献者分散**。bash 工具想加一段自己的使用指引，只能去改中心模板——插件失去自治，模板成为所有人的合并冲突现场。
- **顺序脆弱**。段落顺序影响模型行为（规则应该在工具目录之前还是之后？），靠数组下标或拼接次序维护，插件加载顺序一变，prompt 就悄悄重排。
- **动态与缓存打架**。prompt 里有会变的部分（审批策略、当前目录），每变一次整个 prompt 前缀就变，provider 侧的 prompt cache 全部失效。
- **per-agent 差异无处安放**。subagent 需要不同的 persona 和工具面，中心模板只能靠参数化——参数越加越多，最后变成第二个插件系统。
- **工具目录漂移**。prompt 里的工具列表和注册表的真实状态由两条代码路径维护，第 18 章已经说过这种漂移的代价。

`SystemPrompt` 服务把这五个问题合并成一个答案：一个按 scope 分层、按 order 排序、每次 assembly 现算的注册表。

## 源码剖析

### section 注册：命名、排序、可逆

```ts
// packages/core/system-prompt/src/index.ts
export interface PromptSection {
  /** Unique name — a duplicate registration throws. */
  readonly name: string
  /**
   * Sections are concatenated in ascending order. Convention: `-100` is the
   * harness identity, `0` the deployment persona, tool guidance uses 100–199;
   * other negative orders also render before the persona.
   */
  readonly order: number
  /** Static text or a provider evaluated at each assembly. */
  readonly text: string | ((context: AssembleContext) => string)
  /** Treat this contribution as the complete system prompt. */
  readonly complete?: boolean
}

section(section: PromptSection): () => void {
  if (!Number.isFinite(section.order)) {
    throw new TypeError(`prompt section "${section.name}" order must be a finite number`)
  }
  return this.layers.effect(
    this.ctx,
    layer => layer.sections.insert(section.name, section),
    { label: 'systemPrompt.section()' },
  )
}
```

和 `ctx.tools` 一样，注册走 `ScopedLayers.effect`：返回精确 disposer，插件卸载时段落自动消失；同一层内重名注册抛错，而**跨层重名是特性**——scoped 注册遮蔽全局同名段落。order 是显式数字而非注册顺序，约定成文写在 JSDoc 里：`-100` 是 harness 身份，`0` 是部署 persona，`100–199` 留给各工具的使用指引。服务自己在构造函数里占住前两个槽位：

```ts
// packages/core/system-prompt/src/index.ts
constructor(ctx: Context, config: Config) {
  super(ctx, 'systemPrompt')
  this.toolOrder = validateToolOrder(config.toolOrder)
  if (config.includeHarnessIdentity ?? true) {
    this.section({
      name: 'harness:identity',
      order: -100,
      text: 'You are an AI agent powered by DeepSeek Harness.',
    })
  }
  this.section({
    name: PERSONA_SECTION,   // 'deployment:persona'
    order: PERSONA_ORDER,    // 0
    text: config.persona ?? '',
  })
  // ...
}
```

`PERSONA_SECTION` 被导出为常量不是装饰——`packages/preset/persona` 这个 row 专门 import 它，在 agent scope 里注册**同名同 order** 的段落来遮蔽部署 persona。它的源码注释把理由讲透了：「Imported rather than restated: the registry declares the slot this row replaces, and two hardcoded copies would drift into a preset whose persona silently lands beside the deployment's instead of shadowing it.」——槽位替换之所以是替换而不是重复，靠的就是双方引用同一个常量。这也是 subagent 换 persona 的同一条通路。

### 四种贡献物，一次 assembly

`SystemPrompt` 其实管四张表：`sections`（系统提示段落）、`contexts`（动态运行时上下文，物化为 user-role 快照而非 system prompt）、`toolProviders`（工具 schema 提供者）、`variables`（`{{name}}` 插值变量）。第 20 章见过 `contexts` 的用户：审批服务把策略叙述注册为 order 115 的 context，注释点明动机——「The complete current value travels after retained history, so switching policy does not rewrite the stable system-prompt cache prefix」。**会变的事实进 context（跟在历史后面走），稳定的规则进 section（占住可缓存的前缀）**，prompt cache 的命中率是这个二分法的直接受益者。

`assemble()` 把四张表在一次调用里解析完：

```ts
// packages/core/system-prompt/src/index.ts（删节）
async assemble(context: AssembleContext = {}): Promise<PromptAssembly> {
  const scope = context.scope
  const scopeLayers = this.layers.chainLayers(scope)
  // Scoped variables shadow globals（先全局，再沿链由远及近覆盖）
  const variables: Record<string, string | undefined> = {}
  for (const [name, provider] of this.layers.global.variables.entries()) variables[name] = provider(context)
  for (const layer of scopeLayers) {
    for (const [name, provider] of layer.variables.entries()) variables[name] = provider(context)
  }
  const sectionByName = this.layers.merge(scope, layer => layer.sections)
  // ... contexts 同理；toolProviders 是全部叠加而非遮蔽
  const collected: ToolSchema[] = []
  const knownNames = new Set<string>()
  for (const provider of providers) {
    const result = provider(context)
    collected.push(...result.schemas.map(s => ({ ...s, parameters: structuredClone(s.parameters) })))
    for (const name of result.knownNames ?? /* schema names */) knownNames.add(name)
  }
  const sectionDefinitions = [...sectionByName.values()].sort((a, b) => a.order - b.order)
  // complete section：多于一个则抛错，记住唯一那个
  const assembly: PromptAssembly = {
    sections, contexts, tools: orderTools(collected, this.toolOrder, knownNames), variables,
  }
  const transformed = await this.ctx.waterfall(
    scopeTarget(this, scope), 'system-prompt/assemble', assembly, context,
    () => Promise.resolve(assembly),
  )
  if (completeSection === undefined && !runtimeContextSuppressed) return transformed
  return {
    ...transformed,
    sections: completeSection === undefined ? transformed.sections : [completeSection],
    // ...
  }
}
```

值得注意的三个点。**遮蔽与叠加的区分**：sections/contexts/variables 按名字遮蔽（每个名字一个答案），toolProviders 全部叠加（多个来源的 schema 并存）——语义决定合并策略，不是一刀切。**assembly 后置 waterfall**：`system-prompt/assemble` 让专家插件在成品上做最后变换（返回值即权威），但 `complete` section 在 waterfall **之后**被恢复为唯一段落——一个声明了「我就是完整 prompt」的 persona，连 waterfall listener 也不能往它旁边塞私货，声明的强度由服务兜底。**每次现算**：所有 provider 都以本次 `AssembleContext`（scope + signal，`dsh-agent` 通过 merge-extension 再加上 `agent` 字段）求值，没有缓存的中间状态可以过期。

### 拼装顺序与稳定性

工具列表的排序单独成函数，两条防漂移的措施都在里面：

```ts
// packages/core/system-prompt/src/index.ts
function orderTools(tools: ToolSchema[], toolOrder: string[] | undefined, knownNames: ReadonlySet<string>): ToolSchema[] {
  // ...
  if (toolOrder === undefined) return tools.sort(compareToolNames)
  const unknown = toolOrder.filter(name => name !== TOOL_ORDER_REST && !knownNames.has(name))
  if (unknown.length > 0) {
    throw new Error(`toolOrder lists unregistered tool${/* ... */} known tools: ${[...knownNames].sort().join(', ')}`)
  }
  const listed = new Set(toolOrder)
  const rest = tools.filter(tool => !listed.has(tool.name)).sort(compareToolNames)
  return toolOrder.flatMap(name => name === TOOL_ORDER_REST ? rest : tools.filter(tool => tool.name === name))
}

/** Lexicographic (code-unit) name comparison — locale-independent, so the order is identical on every machine. */
function compareToolNames(a: ToolSchema, b: ToolSchema): number {
  return a.name < b.name ? -1 : a.name > b.name ? 1 : 0
}
```

默认排序刻意用 code-unit 比较而不是 `localeCompare`——注释一句话说明动机：locale 无关，任何机器上的顺序一致。这不是洁癖：工具顺序进 prompt，prompt 进 provider 缓存键，一台机器上多一个 locale 差异就是全量 cache miss 加不可复现的行为差异。配置 `toolOrder` 时必须显式写 `<unlisted-tools>` 占位符（`TOOL_ORDER_REST`）——未列出的工具插到哪里必须是决定，不能是默认；列出未注册的名字在 assembly 时抛错并附上全部已知名字，而「注册过但被当前 scope 限制隐藏」的名字合法缺席——区分这两者正是第 18 章 `knownNames`（限制前名字全集）单独返回的原因。

变量插值同样严格：`{{name}}` 必须是完整的简单引用，格式非法、名字未注册、值为 `undefined` 三种情况全部抛错，绝不静默留下 `{{workspace_root}}` 字样给模型看；替换值不再被二次扫描（没有注入放大）。渲染时空段落被丢弃，其余以空行连接——一个返回 `''` 的条件段落（如 Code Mode 的 collapse 声明在 native scope 下）零成本消失。

### 工具 schema 与 prompt：同一次组装

第 18 章已经给出接线：`ToolRuntime` 构造时 `ctx.systemPrompt.tools(context => this.wireSchemas(context.scope))`。这里补上它的意义：`PromptAssembly` 是 `{ sections, contexts, tools, variables }` 四位一体的结构——**工具 schema 不是与 prompt 平行的另一条通道，而是同一次 assembly 的一个字段**。agent loop 每个 step 调一次 `assemble({ scope, agent, signal })`，拿到的 sections 渲染成 system prompt、contexts 渲染成运行时快照、tools 直接作为请求的工具数组——三者出自同一瞬间的同一批 provider，不存在「prompt 说有这个工具而请求里没带 schema」的中间态。Code Mode 是这个一致性的最佳测试：`tools` 字段收缩到只剩 `run_code` 时，`tools:code-only` 段落（order 99，压在工具指引带之前）与 `tools:sdk` 段落（生成的 SDK 文本）在**同一次** assembly 里出现，且 collapse 段落的渲染条件与执行器的拒绝谓词是同一个 `modeFor(scope)`——prompt 声明的规则和 registry 执行的规则由共享代码保证不劈叉。

### 「prompt 也是插件贡献的」意味着什么

把这一章与前三章连起来看，一个功能插件的全部模型面——工具 schema（`ctx.tools.register`）、使用指引（`ctx.systemPrompt.section`，order 100–199）、动态状态（`ctx.systemPrompt.context`）、执行策略（三段 waterfall）——都通过注册表贡献，都返回 disposer，都按 scope 分层。于是「加一个工具」是纯增量操作：不改中心模板、不碰 agent loop、卸载即整体消失（HMR 场景下 prompt 跟着插件热更新）；「造一个特化 agent」是组合操作：preset 里放一个 persona row、几条 `tools.restrict`、几个 scoped section，宿主代码零改动。prompt 从「一段被所有人编辑的文本」变成「一组有主人、有顺序、有生命周期的声明」——这是插件架构从执行面延伸到模型面的最后一块拼图。

## 设计亮点

> 💎 **设计亮点：order 是公共约定，槽位是导出常量。** 段落顺序不由注册时机决定，而由显式 order 决定，且关键槽位（`PERSONA_SECTION` + `PERSONA_ORDER`）导出为常量供替换者 import。对比普通写法（拼接顺序 = 数组顺序，替换 = 复制粘贴名字），这里插件加载顺序的抖动不影响 prompt，而「遮蔽还是并存」这个语义差异被一个共享常量钉死——想替换 persona 的人不可能拼错槽位名。

> 💎 **设计亮点：稳定规则进 section，易变事实进 context。** 审批策略、沙箱模式这类会话中会变的事实不写进 system prompt，而是注册为跟在保留历史之后的运行时快照。变更时追加新快照即可，请求头的稳定前缀一个字节不动——prompt cache 的失效边界被架构性地控制在「真正变了的部分」。这个二分法贯穿所有内置插件，是缓存友好性从优化手段升格为设计约束的例子。

> 💎 **设计亮点：确定性排序作为跨机器不变量。** 默认工具序用 code-unit 比较（拒绝 `localeCompare`），配置序强制显式 `<unlisted-tools>` 占位并 fail-loud 校验未知名。两台机器、两次运行、两个 locale 下 assembly 出的 prompt 逐字节一致——可复现性不靠祈祷，靠把每一处非确定性来源逐个摁死。

> 💎 **设计亮点：`complete` 在 waterfall 之后恢复。** 「这个 persona 就是完整 prompt」是一个强声明，而 assembly waterfall 又承诺 listener 的返回值权威——两个承诺冲突时，服务的选择是让 waterfall 照常跑（工具、变量、context 仍需解析），然后把 complete 段落恢复为唯一 section。扩展点保持开放，强声明保持不可稀释，各让一步且都有兜底。

## 小结与延伸

`ctx.systemPrompt` 用一张四表注册表（sections/contexts/tools/variables）替换了中心模板：贡献按 scope 分层遮蔽、按显式 order 确定性排序、每 step 现算、经 waterfall 精修再由 complete 声明兜底，工具 schema 与 prompt 文本出自同一次 assembly。它让插件的模型面与执行面获得同样的自治与可逆性，也让 per-agent 特化变成纯组合问题。至此工具系统四章闭环：注册（18）、执行（19）、许可（20）、告知（21）。

延伸阅读：

- `docs/subsystems/system-prompt.md` — 跨包类型契约的官方记录
- `packages/core/system-prompt/tests/tool-order.spec.ts`、`tests/scoped.spec.ts` — 排序与遮蔽的行为契约
- `packages/preset/persona/src/index.ts` — 68 行的槽位替换教科书
- `packages/core/tools/src/code-mode.ts` — `tools:sdk` 段落的 SDK 文本如何生成
- [第 14 章](../part4/14-derive-messages.md) — contexts 渲染的快照如何进入模型历史
