# 第 36 章 Web Client：围绕 session/event 渲染的前端

[上一章](35-web-server-and-api.md)把会话事件送到了浏览器，本章看它们如何变成 UI。`packages/client/` 是仓库里最大的一片（536 个 ts 文件、三十多个 `ui-*` 包），但它的骨架可以用一句话概括：**浏览器里跑着一个真正的 Cordis，前端插件消费与持久层同一套 SessionEvent，经一个增量状态机引擎折叠成 keyed 节点，再由 keyed renderer 逐个渲染**。读懂 `ConversationNodeDefinition` 和 keyed renderer 这一对机制，就读懂了整个前端；`docs/cookbook/adding-a-conversation-node.md` 则展示了第三方如何沿同一条路加一种新节点。

## 问题背景

给 agent harness 写前端，朴素做法是：一个构建期打包的 SPA，Redux/Context 全局 store，每个功能自己 fetch 自己的 REST 资源，聊天记录是一个 `messages` 数组 map 出来的组件列表。三个坑会先后出现：

1. **前后端组合脱节**。服务端是插件化的——某个部署禁用了 goal 插件，但 SPA 是构建期定死的，UI 上 goal 面板照样在，点了报 404。
2. **重渲染风暴**。流式输出下每个 delta 都更新 `messages` 数组，整个列表重渲染；几百个节点的长会话直接卡死，然后开始加 `memo`/虚拟化补丁。
3. **渲染逻辑与事件语义纠缠**。工具调用的「开始/结果/子调度」分散在多种事件里，组件里堆满 `if (msg.type === ...)`；翻页加载旧历史时，半开状态的工具调用如何与新加载的事件缝合，没有系统性答案。

DeepSeek Harness 的前端对这三个坑各有一个结构性回答：客户端模块图是服务端插件树的**实时投影**；每个聊天节点**只订阅自己**；事件到节点的折叠被抽成一个显式的状态机契约 `ConversationNodeDefinition`。

## 源码剖析

### 模块体系：服务端插件树的实时投影

`docs/subsystems/client-modules.md` 描述的机制分三段。首先，一个包通过 `package.json` 声明自己有「客户端一半」（`packages/client/ui-tool/package.json`）：

```json
"dsh": {
  "client": {
    "inject": [
      "@deepseek-ai/dsh-client-runtime",
      "@deepseek-ai/dsh-client-locale",
      "@deepseek-ai/dsh-client-ui-conversation"
    ],
    "platform": "web"
  }
}
```

其次，Host 侧的 `ClientModuleRegistry`（`packages/client/modules/src/index.ts`，service 名 `clientModules`）**增量扫描 Loader 的存活 entry**：每次 `internal/plugin` fiber 构建/销毁把对应包标脏，microtask 批量对账，组合出 `WebBootGraph` 注入 `index.html` 的 `window.__DSH_BOOT__`，并按 `GET /plugins/<id>/client.js` 提供 bundle。关键在于：**只有真正存在于当前服务端插件树里的包才会被下发客户端 bundle**。在 `cordis.patch.yml` 里 disable 一个服务端插件，下一次刷新它贡献的所有 UI 面随之消失——不需要任何手工同步，服务端与客户端的组合是同一个 artifact 的两个视角。

最后，浏览器侧 `ClientModuleSystem`（`packages/client/modules/src/client/system.ts`）实现了一个跑在 `<script>` 标签上的惰性 CJS：

```ts
// packages/client/modules/src/client/system.ts
private makeRequire(edges: Set<string>): (spec: string) => unknown {
  return (spec: string): unknown => {
    edges.add(spec)
    if (this.seed.has(spec)) return this.seed.get(spec)
    if (this.statics.has(spec)) return this.statics.get(spec)
    const id = stripClientSuffix(spec)
    const record = this.loadCache.get(id)
    if (record !== undefined) return record.exports
    if (this.factories.has(id)) return this.materialize(id).exports
    throw new Error(
      `client-modules: require("${spec}") missed the module table — not a platform seed word, not a shell-own module, `
      + 'and no registered factory (a build-time externals drift, or a forbidden cross-plugin value import)',
    )
  }
}
```

而它服务的对象，是**浏览器里的 Cordis Loader**。`packages/client/web/src/boot.tsx`：

```tsx
// packages/client/web/src/boot.tsx
private async runPluginBoot(prefetching: Promise<void>): Promise<void> {
  const ctx = this.ctx
  await ctx.plugin(Loader)
  const loader = ctx.loader
  // Inject the module system BEFORE any entry exists: tree.import falls back
  // to a bare dynamic import when internal is undefined, which in a browser
  // is a guaranteed loud failure — correct as a tripwire, never as a path.
  loader.internal = this.modules as never
```

`Loader.internal` 本是为 Node 文件系统解析设计的接缝，这里被指向浏览器模块系统——于是每个 `WebBootEntry` 成为一个 Loader entry，客户端扩展包就是**如假包换的 Cordis 插件**：`apply(ctx)`、`inject` 等待、fiber 生命周期、fail-loud 激活扫描，与服务端（[第 34 章](34-boot.md)）语义完全一致。前端不是「借鉴」了服务端的插件架构，它**就是**同一个架构跨过了进程边界。

### 会话渲染管线：从事件到 keyed 节点

WS 帧进入后的完整流水线是：`ConnectionController` → `SessionManager.handleMuxEnvelope`（`packages/client/runtime/src/client/sessions/manager.ts`）→ `Session.appendLive` → `ConversationNodeAssembler.append` → 各 `ConversationNodeDefinition` → `ChatSnapshot` → zustand store → React。核心契约在 `packages/client/runtime/src/client/contract/conversation.ts`：

```ts
// packages/client/runtime/src/client/contract/conversation.ts
export interface ConversationNodeDefinition<State = unknown> {
  readonly kind: string
  /** Sole view target owned by this Definition; omitted for state-only Contexts. */
  readonly target?: string
  match(event: SessionEvent): ConversationMatchResult | null
  start(context: ConversationNodeContext<State>, match: ConversationMatch,
    reader: ConversationContextReader): State
  update(context: ConversationNodeContext<State> & { readonly state: State },
    match: ConversationMatch): State
  publication?(match: ConversationMatch): ConversationPublication
  buildLocationData?(context: ConversationNodeContext<State>,
    scope: ConversationLocationDataScope): ConversationLocationData | null
  buildViewNode?(context: ConversationNodeContext<State>): ConversationViewNode | null
}
```

这是一个「事件族 → 节点」的显式状态机：`match` 是纯函数，从单个事件提取业务身份（如 `callId`）和角色（start/update），**无权访问历史**；引擎按 `conversationContextKey(kind, id)` 为每个身份维护一个 Context，`start` 建初态、`update` 按日志序吸收后续事件、`buildViewNode` 物化出渲染节点。看真实的工具调用定义（`packages/client/ui-conversation/src/client/conversation-nodes/tool.ts`，节选）：

```ts
// packages/client/ui-conversation/src/client/conversation-nodes/tool.ts
export const toolDefinition: ConversationNodeDefinition<ToolState> = {
  kind: 'tool-call',
  target: 'chat',
  match: (event) => {
    if (event.type === 'tool/call') return { id: String(event.data.callId), role: 'start' }
    if (event.type === 'tool/result' && isAppendSurfaceEvent(event)) {
      return { id: String(event.data.message.source.callId), role: 'update' }
    }
    // ... tool/code-dispatch* 事件按 rootCallId 归并到同一 Context
    return null
  },
  start: (_context, match) => ({ root: rootCall(match), children: new Map(), parents: new Map() }),
  // ...
}
```

`tool/call`、`tool/result`、`tool/code-dispatch` 三种事件被 `callId` 收拢进同一个 Context——组件里再也没有跨事件的 `if` 拼图。汇编器（`conversation-assembler.ts`）为三种摄入路径维持**结构上不同的成本曲线**：`replaceWindow` 全量重建；`prepend`（翻页加载旧历史）保留既有 Context/节点身份、只重放受影响的 Context；`append`（实时尾部）严格 O(定义数) 次 match + O(1) Context 查找，**禁止扫描历史或其他 Context**。`reader.previous<State>(kind)` 允许 `start` 里查询更早的前驱 Context，引擎记录这条依赖，翻页导致前驱变化时自动重跑依赖者——一个为「重放 append-only 日志」量身定做的手写增量数据流图。

### keyed renderer：节点只认自己的 key

物化出的节点进入 zustand snapshot 后，`ChatView` 只 map `snapshot.chat.order` 渲染一列 seat，每个 seat 只订阅一个 key（`packages/client/ui-conversation/src/client/chat/ChatNodeSeat.tsx`）：

```tsx
// packages/client/ui-conversation/src/client/chat/ChatNodeSeat.tsx
/** Subscribe and dispatch one stable Context key without observing sibling Nodes. */
export const ChatNodeSeat = memo(function ChatNodeSeat({
  nodeKey, selectedCallId, cwd, openFile, inspectCall, forkAt,
  loadImage, fileMentions, useSession, renderSlot, t,
}: ChatNodeSeatProps) {
  const node = useSession(snapshot => snapshot.chat.nodes.get(nodeKey))
```

`useSession` 基于 `useSyncExternalStoreWithSelector`（整个客户端栈唯一的 hook 构造器 `bindSnapshotSelector`，在 `packages/client/web-react/src/bind.ts`）：流式 delta 只改 Map 里一个 entry，只有那一个 seat 的 selector 触发。节点到组件的分发走 keyed slot：渲染器按 `kind` 注册（`packages/client/ui-conversation/src/client/chat/register-node-renderers.ts`）：

```ts
// packages/client/ui-conversation/src/client/chat/register-node-renderers.ts
ctx.slots.inject('conversation.chat.node', () => ctx.slots.register(
  { name: 'conversation.chat.node', key: 'user', locale: NS }, UserMessageNodeView))
ctx.slots.inject('conversation.chat.node', () => ctx.slots.register(
  { name: 'conversation.chat.node', key: 'assistant-step', locale: NS }, AssistantNodeView))
```

seat 端 `renderSlot('conversation.chat.node', owner, { entryKey: node.kind })`，slot 宿主按 `options.key === entryKey` 做 O(1) 查表分发（`packages/client/web-react/src/scoped-slots.tsx`）。Definition 和它的渲染器可以住在**不同的包**里——`tool-call` 的状态机在 ui-conversation，React 组件在 ui-tool 注册——耦合只剩一个 `kind` 字符串，而这个字符串又被类型系统看住（见下）。

### 扩展路径与三个值得借鉴的决策

`docs/cookbook/adding-a-conversation-node.md` 的完整扩展路径，浓缩为五步：① 设计可重放的事件族——每个事件携带同一个稳定业务 id，并通过 `declare module '@deepseek-ai/dsh-session/types'` 合并进 `SessionEventMap`；② 在客户端插件实现 `ConversationNodeDefinition`，并用 declaration merging 声明 Chat payload（`interface ChatNodeDataMap { 'review-job': ReviewChatData }`）；③ 两半分别注册：`ctx.conversationEvents.register(definition)` + keyed slot 渲染器；④ 需要跨节点信息时用 `reader.previous`（仅限 `start`）或 `buildLocationData` 发布到 Turn/Step；⑤ 验证重放不变量——全量替换 ≡ 翻页重放 ≡ 实时追加。

值得单独借鉴的架构决策有三个。

**状态管理：引擎无 React，React 只有一个绑定点。** 状态引擎是 `zustand/vanilla` + `subscribeWithSelector` + immer `produce` + 手写 `rafBatch`（默认同步 flush，高频流式更新可选 raf 批量），定义在 React-free 的 `packages/client/runtime/src/client/contract/store.ts`；React 绑定被压缩到 `web-react/bind.ts` 一个函数。服务端状态（事件重放出的 `ConversationSnapshot`）与 UI 本地状态（选中项、草稿、主题——`defineStore` 声明、可选持久化）是结构上分开的两条轴，前者永远不进 React state、不做客户端持久化。

**协议同构：类型跨进程靠 declaration merging，不靠生成层。** `SessionEventMap` 由**服务端的**事件生产包声明，客户端 Definition 以 type-only import 消费同一个类型；`ChatNodeDataMap` 等是框架包里的空接口座位，每个业务包用 `declare module` 追加自己的 kind → payload 映射。`ChatNode<Kind>` 因此能按 `node.kind` 正确收窄，新增业务包在类型层面和在 Cordis 插件层面同样是**纯增量**的——没有任何中心注册文件需要改，运行时再用 `node.key !== context.key` throw 作为兜底断言。

**摄入路径的成本不对称是构造出来的，不是约定出来的。** live append 路径拿不到扫描历史的 API——`match` 的签名里只有一个事件。想写出 O(n) 的实时路径，类型系统先拦住你。

## 设计亮点

> 💎 **设计亮点：Cordis 原封不动跑进浏览器**
> 常规做法是前端另起一套模块/DI 体系，与服务端插件架构「精神一致」。这里直接把 `Loader.internal` 接缝从 Node 文件系统解析换成手写的惰性 CJS 模块系统，客户端扩展就是真 Cordis 插件——同一套 `apply/inject`、同一套 fiber 生命周期、同一套 fail-loud 审计。学一次插件模型，两端通用；服务端章节讲过的每条纪律在前端原样成立。

> 💎 **设计亮点：客户端模块图是服务端组合的实时投影**
> `ClientModuleRegistry` 扫的不是静态清单，是 Host Loader **当前存活的 fiber**。禁用一个服务端插件，它的 UI 从下一次页面加载起整体消失。「前后端功能开关不同步」这类 bug 在结构上不存在——因为不存在两份需要同步的东西。

> 💎 **设计亮点：每个节点只订阅自己，重渲染风暴无处发生**
> `order` 数组与节点 Map 分离，`ChatNodeSeat` 用 selector 只订阅自己的 `nodeKey`，`kind` 分发是 O(1) 查表。几百节点的会话里，流式 delta 只让一个 seat 重渲染——不是靠事后加 memo 补救，而是订阅粒度从架构上就等于更新粒度。

> 💎 **设计亮点：把「事件→UI」抽成显式状态机契约**
> `match/start/update/buildViewNode` 把跨事件的拼装逻辑从组件里连根拔出，还顺带解决了最难的翻页缝合问题：prepend 保身份、只重放受影响 Context，`reader.previous` 的依赖被引擎记录并自动重跑。渲染器退化成纯函数：`node.data` 进，JSX 出。

## 小结与延伸

Web Client 的三层结构——实时投影的模块系统、事件到节点的增量汇编器、keyed 订阅与分发——共同回答了同一个问题：如何让前端与「会话即事件日志」（[第 13 章](../part4/13-session-event-log.md)）的世界观保持同构。前端消费的类型就是服务端记录的类型，前端的组合就是服务端的组合，前端的扩展方式就是服务端的扩展方式。下一章离开浏览器，看[没有 server 的嵌入路径](37-sdk-and-headless.md)。

延伸阅读：

- `docs/subsystems/client-modules.md`、`docs/subsystems/web.md`、`docs/web-styling.md` — 模块体系与样式约定
- `docs/cookbook/adding-a-conversation-node.md` — 端到端扩展演练（含验证清单）
- `packages/client/runtime/src/client/contract/conversation.ts` — Definition 契约全文
- `packages/client/runtime/src/client/sessions/conversation-assembler.ts` — 三种摄入路径的汇编器
- `packages/client/ui-conversation/src/client/conversation-nodes/` — assistant/tool/inbox 等真实 Definition
- `packages/client/modules/src/client/system.ts`、`packages/client/web/src/boot.tsx` — 浏览器模块系统与浏览器内 Cordis boot
- `packages/client/runtime/src/client/contract/store.ts`、`packages/client/web-react/src/bind.ts` — 状态引擎与唯一的 React 绑定点
