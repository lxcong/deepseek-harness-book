# 附录 A：术语表

本表以仓库 `docs/glossary.md` 为底本，结合 `docs/cordis-primer.md` 与 `docs/architecture.md` 中的规范定义，按主题重新分组。每条术语给出中文定义，并链接到本书首次系统讲解它的章节。仓库的原则是"一个概念只用一个规范术语"，本表沿用同一纪律：术语本身保留英文原文。

## Cordis 概念

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **plugin** | 实现 Service 协议的对象：可以是带可选 `inject` 与 `apply(ctx)` 的函数，也可以是被 Cordis 挂载进当前 context 的 `Service` 子类。dsh 中一切皆插件。 | [第 3 章](../part1/03-architecture-overview.md) |
| **Context（`ctx`）** | 服务的仓库。服务向 context 认领一个稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件按 key 查找服务，而不是 import 具体实现。 | [第 4 章](../part2/04-context-and-service.md) |
| **Service** | 认领 `ctx.<key>` 并拥有自己词汇类型的 Cordis 类。接缝的 Service Definition 必须是抽象类或具体注册表，从不是 TypeScript `interface`。 | [第 4 章](../part2/04-context-and-service.md) |
| **`inject`** | 插件声明所需服务的方式：命名了依赖的插件会等到这些服务存在才加载，加载顺序由服务依赖表达，而非手工排 boot 序。 | [第 4 章](../part2/04-context-and-service.md) |
| **effect（`ctx.effect()`）** | 可逆副作用：prompt section、tool schema、adapter、listener 等注册全部通过 `ctx.effect()` / `ctx.on()` 安装，reload 与 teardown 时可预测地回退。每个注册都应有 disposer。 | [第 6 章](../part2/06-reversible-effects.md) |
| **Loader** | 解析配置、挂载插件树的 Cordis 组件；`!!js` 表达式节点、`disabled` 字段的插值都由它处理。 | [第 7 章](../part2/07-profiles-and-bundles.md) |
| **profile** | 存放于 Harness home 的具名组合：列出叠放的 bundle、持有 out-of-tree 插件与用户自己的 `cordis.patch.yml`。`web` 与 `headless` 作为模板随发行。 | [第 7 章](../part2/07-profiles-and-bundles.md) |
| **bundle** | Cordis 配置行及其挂载代码的分发格式；它插入的内容始终可被上层 patch。`dsh-base` 是每个 profile 的第一层。 | [第 7 章](../part2/07-profiles-and-bundles.md) |
| **patch（`cordis.patch.yml`）** | 按 id 定位一行配置并整体替换其 config、或插入新行的覆盖层。图层顺序：各 bundle → profile patch → home patch → `--patch` overlay。 | [第 7 章](../part2/07-profiles-and-bundles.md) |
| **vendoring** | dsh 把 Cordis 整个搬进 `vendor/` 随仓库演进的做法，同步流程见 `vendor/README.md`。 | [第 42 章](../part10/42-vendoring.md) |

## 分发模式（Dispatch Modes）

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **emit** | 不等待、无返回值的广播；listener 按注册顺序观察。 | [第 5 章](../part2/05-typed-events.md) |
| **waterfall** | around-middleware 式分发：listener 收到 `(...args, next)`，调用 `next()` 委托下游（可包装结果），不调用则短路。有返回值，不整体等待。 | [第 5 章](../part2/05-typed-events.md) |
| **parallel** | 所有 listener 并行观察事件，整体等待完成，无返回值（如 `session/flush`）。 | [第 5 章](../part2/05-typed-events.md) |
| **serial** | listener 按注册顺序依次执行并被等待，有返回值、无 `next()`（如 `agent/turn-stopping`）。 | [第 5 章](../part2/05-typed-events.md) |
| **short-circuit** | waterfall 中 policy listener 不调用 `next()` 直接返回、独占决策的设计；仅做注解或观察的 listener 必须委托。 | [第 5 章](../part2/05-typed-events.md) |
| **`@mode` 标签** | 事件的 dispatch mode 是公开契约的一部分，新事件用 `@mode` 文档标签声明，生成式目录会校验声明与实际分发点一致。 | [第 40 章](../part10/40-docs-as-engineering.md) |

## Agent 循环

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **turn** | 一次对 session 已受理输入的排空（drain）：在第一份输入被 claim 前开启，在模型及其工具停下或终止策略介入后关闭。一个 turn 含零个或多个 step。 | [第 8 章](../part3/08-turn-and-step.md) |
| **step** | 一次模型请求加上其响应引发的工具执行。 | [第 8 章](../part3/08-turn-and-step.md) |
| **round** | 包含一个 turn 的外层策略迭代（如 goal round、一次 Ralph 尝试）。round 计数属于该策略，不统计会话里的每个 turn。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| **inbox** | 输入到达 driver 的唯一通道。有的消息立刻唤醒循环；注入的上下文（injected context）留在 inbox 里，等其他消息一起被 claim。 | [第 10 章](../part3/10-inbox-and-injection.md) |
| **`agent.inject()`** | 添加 model-facing 上下文的入口：注入的内容落入下一个被受理的请求。 | [第 10 章](../part3/10-inbox-and-injection.md) |
| **`agent/pre-step`** | 决定模型看到什么的 waterfall：listener 可改写被 claim 的消息或直接 reject。被 reject 或首个 enter 改写为空的 claim 仍会关闭一个未消耗 step 的持久 turn，日志因此记下这次尝试。 | [第 9 章](../part3/09-agent-loop-deep-dive.md) |
| **scope** | per-agent 注册的单位：一项贡献（工具、prompt section、变量、restriction、listener）要么 global（所有 agent 可见），要么 scoped（归属恰好一个 scope key）。两层、扁平，不向 subagent 继承。 | [第 11 章](../part3/11-scope.md) |
| **scope key** | scope 的不透明身份，按对象同一性比较。惯例：活跃的 agent 就是自己 scope 的 key。 | [第 11 章](../part3/11-scope.md) |
| **agent context（`agent.ctx`）** | agent 的 scoped context：经它做的注册"scope 可见且 scope 同寿命"（一个事实驱动两件事），其上的 listener 参与该 agent 的 scope 过滤分发。 | [第 11 章](../part3/11-scope.md) |
| **scope carrier / scoped dispatch** | scope 过滤分发携带的 `thisArg`（由 `scopeTarget` 构造）；关于某个 agent 活动的事件用该 agent 的 carrier 分发，关于注册表自身的事件（registry-subject）保持不过滤。 | [第 11 章](../part3/11-scope.md) |
| **shadowing** | 最具体者胜的名字解析：scoped 的工具/section/变量仅对本 scope 替换同名 global 版本。per-agent 人格与工具变体的机制。 | [第 11 章](../part3/11-scope.md) |
| **restriction（`tools.restrict`）** | 对一个 scope 过滤 GLOBAL 工具集（多条按交集组合）；scope-local 注册在过滤之后合入。被滤掉的 global 工具在 prompt 中缺席且拒绝执行，与不存在的工具不可区分。 | [第 20 章](../part5/20-approval-and-presets.md) |
| **setup window** | agent 创建槽位（`CreateAgentOptions.setup`）：scope 与 agent 对象已存在、但 agent/session 尚未发布、`agent/session-start` 尚未触发、首个 prompt 尚未组装之前。setup 只注册，从不驱动 agent。 | [第 11 章](../part3/11-scope.md) |
| **lineage** | 以数据承载的父子事实（`parentSession`、持久的 `delegationDepth`、运行时的 `subagentDepth`）；从不影响可见性。 | [第 30 章](../part8/30-subagent.md) |
| **goal** | 附着在既有会话上的唯一持久完成目标，带版本化的 `active` / `paused` / `blocked` / `complete` 阶段与 goal-round 上限。goal 是状态，不是调度器，会话日志仍是其 source of truth。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| **goal round** | 为当前 goal 受理的一次续跑周期，同会话 driver 将其物化为一个 goal 来源的 turn；同会话里无关的人类 turn 不消耗 round 上限。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| **goal activation** | 进程本地的续跑许可（`armed` / `disarmed`），刻意不进入持久回放——resume 和 fork 后必须经 `/goal` 或模型工具的人类授权 resume 才能恢复自动工作。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| **Ralph loop / round / handoff** | 一次朝不可变目标推进的前台 fresh-agent 工作流：每个 round 是不带父辈对话种子的全新子会话，跨 round 状态由共享 workspace 与一份有界结构化 handoff（status、summary、evidence、next steps、blocker）承载。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| **subagent** | 经 `ctx.subagents` 具名 provider 委托的子代理，provider 谱系从进程内 fork/spawn 到 ACP、SDK 乃至另一个产品（Claude Code、Codex）。 | [第 30 章](../part8/30-subagent.md) |

## 会话

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **SessionEvent / session log** | 会话的 append-only 事件日志，经 `session/event` 广播的持久事实。凡必须在重载后幸存的事实都应成为一条 session event。 | [第 13 章](../part4/13-session-event-log.md) |
| **`deriveMessages()`** | 从日志投影模型历史的函数；原始 `assistant/chunk` 事件保障回放与 UI 保真。fork、resume、转录、遥测、持久化全部派生自这条流。 | [第 14 章](../part4/14-derive-messages.md) |
| **"Model-visible means logged"** | 运行时不变量：凡进入模型请求的内容必须可从日志重建。新增 model-visible 输入意味着扩展 `SessionEventMap` 并从日志渲染。 | [第 15 章](../part4/15-model-visible-invariant.md) |
| **fork** | 复制一个活跃会话到子会话：`ctx.sessions.fork(source, boundary?, childSessionId?)`。 | [第 16 章](../part4/16-fork-resume-compaction.md) |
| **compaction** | 压缩上下文的服务接缝（`ctx.compaction`），默认实现由 token-meter 驱动、用 LLM 做摘要。 | [第 16 章](../part4/16-fork-resume-compaction.md) |
| **spill** | 超限工具输出的溢出存储接缝（`ctx.spillStore`）：把超大工具文本存为会话私有文件，模型只保留预览与取回定位符。 | [第 17 章](../part4/17-spill-and-storage.md) |
| **projection** | 从日志推导的 per-session 当前状态（`ctx.sessionProjections`），带可合并扩展的类型表与持久化投影缓存。 | [第 14 章](../part4/14-derive-messages.md) |
| **session persistence** | 持久化接缝（`ctx.sessionPersistence`），JSONL 与 SQLite 两种后端；`session/flush` 以 parallel 模式等待落盘。 | [第 17 章](../part4/17-spill-and-storage.md) |

## 工具

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **tool** | 面向模型的能力：注册在 `ctx.tools` 上，其 schema 加入 prompt 组装，执行走守卫流水线。 | [第 18 章](../part5/18-tool-registry.md) |
| **tool pipeline** | `tools/pre-execute → tools/execute → tools/post-execute` 三段 waterfall：前段做准入与改写，中段包裹执行，后段做结果加工（spill、hooks、提醒等）。 | [第 19 章](../part5/19-tool-pipeline.md) |
| **approval（`ctx.approval`）** | 一次性权限决策接缝：请求经 `approval/request` waterfall 分发给组合的 answerer，默认 fail-closed。 | [第 20 章](../part5/20-approval-and-presets.md) |
| **permission preset** | 面向用户的权限预设（`ctx.permissionPresets`）：一个产品级 Permissions 选择器捆绑 sandbox-mode 与 approval-policy 两个旋钮，写穿为各自的 session event。 | [第 20 章](../part5/20-approval-and-presets.md) |
| **human command** | 斜杠前缀、由人类侧适配器经 `ctx.commands` 解释执行的指令，不会变成模型消息；区别于模型工具与 `ctx.shell` 的命令执行。 | [第 36 章](../part9/36-web-client.md) |
| **command plane** | UI 适配器与命令插件拥有的发现、解析、分发、取消与结果渲染层；命令输出是 UI 状态，除非 handler 另行修改某个持久域。 | [第 36 章](../part9/36-web-client.md) |
| **system prompt section** | 插件注册的 prompt 片段，经 `system-prompt/assemble` waterfall 组装为每个 step 的系统提示。 | [第 21 章](../part5/21-system-prompt.md) |
| **skill** | 经 `ctx.skills` provider 注册表提供的技能包，由模型侧 skill 工具按需加载。 | [第 31 章](../part8/31-skills-and-hooks.md) |
| **hook** | 兼容 Claude Code / Codex hooks.json 线协议的外部拦截脚本，桥接到 dsh 的拦截接缝上。 | [第 31 章](../part8/31-skills-and-hooks.md) |
| **job** | 后台任务注册表（`ctx.jobs`）中的长时工作：共享 id、owner 隔离、轮询、取消，由 `job_*` 工具收取或终止。 | [第 32 章](../part8/32-workflow-jobs-goal.md) |

## 接缝（Capability Seams）

| 术语 | 定义 | 首次出现章节 |
|---|---|---|
| **seam** | 可整体替换的能力，由三角构成：Service Definition（拥有 `ctx.<key>` 与词汇类型）、一个或多个 Service Provider、一个或多个 Consumer。seam 指完整能力，从不指单一角色。 | [第 22 章](../part6/22-seam-triangle.md) |
| **Service Definition** | 声明接口的 Cordis Service（抽象类如 `ShellExecutor`，或具体注册表如 `WebRuntime`），从不是 TypeScript `interface`。 | [第 22 章](../part6/22-seam-triangle.md) |
| **Service Provider** | 实现某个 Definition 的插件，如 `dsh-bash-local` / `dsh-bash-sandbox` 之于 `dsh-shell`。 | [第 22 章](../part6/22-seam-triangle.md) |
| **Consumer** | 注入并使用服务的一方，常见形态是面向模型的工具（如 `dsh-tool-bash`）。 | [第 22 章](../part6/22-seam-triangle.md) |
| **capability events** | 不 import 主循环、直接把策略与适配器挂到接缝上的事件族（`fs/*`、`tools/*` 等），与 session events、agent events 并列为三大事件域。 | [第 5 章](../part2/05-typed-events.md) |
| **execution world** | 文件系统与子进程 provider 共享的同一个执行环境：把它们指向远程沙箱，Bash、PTY、LSP 整体随迁，无需 fork provider。 | [第 24 章](../part6/24-execution-world.md) |
| **sandbox（`ctx.sandbox`）** | 进程约束接缝：同世界的约束词汇与 `SandboxProvider` 契约，本地后端有 bwrap、Landlock、macOS Seatbelt、Windows ACL，功能探测、fail-closed。 | [第 25 章](../part6/25-sandbox.md) |
| **sandbox mode** | per-call 的沙箱模式（由 sandbox-policy 解析）：read-only 拒绝变更，workspace-write 把变更圈进 workspace 与 temp 根。 | [第 25 章](../part6/25-sandbox.md) |
