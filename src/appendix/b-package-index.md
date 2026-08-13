# 附录 B：包索引

本索引遍历 `packages/*/` 下全部 219 个子包，按目录分组。职责一栏译写自各包 `package.json` 的 `description`；`ctx` key 仅列出在描述或架构文档中明确认领的服务键；章节一栏指向本书讲解该子系统的主要章节。包名统一省略 `@deepseek-ai/` 前缀。

## acp/ — Agent Client Protocol

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-acp` | 仅面向自动化的 ACP 服务器，经 JSON-RPC stdio 驱动 dsh agent | — | [第 37 章](../part9/37-sdk-and-headless.md) |

## api/ — 远端 API 装配

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-api-gateway` | Typert Remote Host 分发器与 Client API 端点 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-api-remotes` | Remote BFF 装配与 Host 侧 Agent/Session 查找策略 | — | [第 35 章](../part9/35-web-server-and-api.md) |

## attachment/ — 附件存储

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-attachment` | 持久、不可变的附件存储接缝 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-attachment-local` | DSH_HOME 下私有内容寻址的附件存储实现 | — | [第 17 章](../part4/17-spill-and-storage.md) |

## boot/ — 启动

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-app-boot` | app bin 共享的 boot 胶水：.env 加载、fail-loud Loader 守卫、快照感知的配置解析与 Loader 启动序列 | — | [第 34 章](../part9/34-boot.md) |
| `dsh-cmdline` | 从 dsh 启动器到 app 插件的不可变命令行交接 | — | [第 34 章](../part9/34-boot.md) |

## bundle/ — 发行 bundle

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-base` | 共享核心的 profile bundle：每个 profile 的第一层 patch | — | [第 7 章](../part2/07-profiles-and-bundles.md) |
| `dsh-headless` | 一次性运行 bundle：dsh-base 之上的直连 Agent/Session runner，无 Host/HTTP/浏览器层 | — | [第 7 章](../part2/07-profiles-and-bundles.md) |
| `dsh-web-app` | 浏览器界面 bundle：dsh-base 之上的 web patch 层与运行时胶水插件 | — | [第 7 章](../part2/07-profiles-and-bundles.md) |

## client/ — Web Client（浏览器端插件）

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-client-web` | Web shell 内核：`bootWebShell` 两阶段启动与 app-shell 装配入口 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-connection` | 线缆消费层：HTTP 上行/WebSocket 下行客户端与带重连的双流 `ConnectionController` | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-modules` | 客户端模块系统（双面）：node 半侧组装 `__DSH_BOOT__` 入口图，浏览器半侧是 vendored Loader 消费的惰性 CJS 模块表 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-runtime` | 客户端核心服务：SlotRegistry、SessionRuntime（scope 树 + 对象层） | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-web-react` | Shell 侧 React 胶水：`createSlotRenderer`、`SessionProvider`、快照选择器桥、`useInvoke` | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-hmr` | 仅开发用的热重载驱动：SSE 重建帧 → 失效/预取 → fiber 切换 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-locale` | Locale 插件：Host 存储的 zh/en 偏好、浏览器兜底与类型化命名空间词典 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-schema-form` | 设置编辑器的 schema/draft 模型层：反水化 schemastery、校验并按路径不可变编辑 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-slots` | Slot 注册表纯核心：SlotMap 声明合并、单一 register 组合 API、渲染器安装接缝 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-layout` | 三栏 AppFrame 外壳与 `ctx.layout` 视图状态服务 | `ctx.layout` | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-primitives` | 纯 React 原子组件：控件、图标、markdown、JSON 查看器（零 cordis） | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-conversation` | 会话域：骨架、有序聊天流、composer 与详情宿主 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-sidebar` | 侧栏：会话多级树、搜索、分组、状态圆点 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-tool` | 工具调用树渲染器与按工具键控的展示 slot | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-commands` | 命令界面：全局目录缓存、`/` 触发源、三种命令 UI、popupSelect 注册表 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-input-trigger` | 输入触发流水线：`/` 与 `@` 检测、候选菜单、pick 路由 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-attachment` | 附件原子组件：草稿图片栏、消息图库、原图 lightbox | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-agent-preset` | Agent preset 界面：默认 preset、本会话席位、组合编辑器 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-goal` | 会话 goal 界面：composer 上方停靠的 GoalBar | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-jobs` | 会话头部后台任务列表：镜像 session/jobs 帧的实时注册表状态 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-plan` | Plan 模式 composer 控件与 `/plan` 命令通道 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-model-selection` | 模型选择：`/model` popupSelect | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-permission-presets` | 权限界面：新会话默认值与当前会话 `/permission` 弹窗 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-message-feedback` | 逐消息反馈控件，接 messageFeedback Host Remote | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-deliverables` | 产出文件的 turn 尾栏与最终回复中的可点击文件引用 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-skill` | Web 端技能引用与专用 skill 工具行 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-subagent` | Subagent 会话目录、续跑路由 UI 与 `@` 引用源 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-trajectory` | 轨迹事件账本与交互式时序总览（纯消费者插件） | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-user-questions` | `ask_user_question` 的 Web 侧：工具挂载与 composer 接管问答 UI | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-workflow-run` | 持久 workflow-run 会话节点与嵌套成员展开 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-workspace` | Workspace 选择器：侧栏与空态的 WorkspacePicker | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-theme` | 主题：预插件调色板 Host 引导、light/dark/system 状态、`--dsw-*` token | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-settings` | 设置域基座：settings 命名空间 scope 服务与规范 slot 契约 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-settings-general` | General 设置分区、外壳 chrome 内容与版本化欢迎通知 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-settings-models` | Models 设置与共享的产品引导对话框 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-settings-plugins` | Plugins 设置分区与可配置的 host 侧插件卡片 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-settings-plugin-inventory` | 只读的 Cordis Loader 库存标签页 | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-directory-picker-browse` | 应用内目录浏览界面（渲染 host 的列举/创建原语） | — | [第 36 章](../part9/36-web-client.md) |
| `dsh-client-ui-directory-picker-native` | 原生目录选择界面（驱动 host 的 OS 选择器，无渲染） | — | [第 36 章](../part9/36-web-client.md) |

## code-runtime/ — 代码执行

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-code-runtime` | 抽象代码执行接缝 | `ctx.codeRuntime` | [第 26 章](../part6/26-lsp-and-code-runtime.md) |
| `dsh-code-runtime-worker-thread` | worker-thread 实现 | — | [第 26 章](../part6/26-lsp-and-code-runtime.md) |

## compaction/ — 上下文压缩

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-compaction` | 抽象压缩服务接缝 | `ctx.compaction` | [第 16 章](../part4/16-fork-resume-compaction.md) |
| `dsh-compaction-basic` | token-meter 驱动的压缩策略与 LLM 摘要后端 | — | [第 16 章](../part4/16-fork-resume-compaction.md) |
| `dsh-compaction-tool-result-pruner` | 回放安全、免模型的工具结果头/中/尾裁剪 | — | [第 16 章](../part4/16-fork-resume-compaction.md) |
| `dsh-command-compact` | 显式压缩的人类斜杠命令 | — | [第 16 章](../part4/16-fork-resume-compaction.md) |

## context/ — 上下文注入

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-agent-instructions` | AGENTS.md / CLAUDE.md 指令文件的 workspace 上下文加载器 | — | [第 10 章](../part3/10-inbox-and-injection.md) |
| `dsh-session-reference` | 跨会话快照引用与持久的不可信模型上下文 | `ctx.sessionReferenceResolver` | [第 10 章](../part3/10-inbox-and-injection.md) |
| `dsh-time-context` | 可选的按 step 持久时间上下文（当前时间与耗时） | — | [第 10 章](../part3/10-inbox-and-injection.md) |
| `dsh-tmux-context` | 可选的按 step 持久 tmux 位置上下文 | — | [第 10 章](../part3/10-inbox-and-injection.md) |

## core/ — 核心

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-agent` | Agent 接口、活跃注册表、initiator scope 与 `agent/*` 事件词汇 | `ctx.agents` | [第 8 章](../part3/08-turn-and-step.md) |
| `dsh-agent-loop` | 实现 Agent 接口的默认 driver（具体主循环插件） | `ctx.agentLoop` | [第 9 章](../part3/09-agent-loop-deep-dive.md) |
| `dsh-scope` | scoped-context 注册原语：scope 标签与 scope 过滤分发（库，无 ctx key） | — | [第 11 章](../part3/11-scope.md) |
| `dsh-session` | 事件溯源的会话存储：append-only `SessionEvent` 日志与内存 store | `ctx.sessions` | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-system-prompt` | 系统提示组装注册表 | `ctx.systemPrompt` | [第 21 章](../part5/21-system-prompt.md) |
| `dsh-tools` | 工具注册表与守卫执行流水线 | `ctx.tools` | [第 18 章](../part5/18-tool-registry.md) |
| `dsh-agent-default-model` | Agent 入口共享的默认模型选择 | — | [第 27 章](../part7/27-llm-vocabulary-and-streaming.md) |
| `dsh-agent-tool-presentation` | Agent 面工具呈现选择器：Code Mode、原生或两者 | — | [第 18 章](../part5/18-tool-registry.md) |

## credentials/ — 凭证

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-credentials` | 抽象凭证接缝：settings 只存引用，provider 持有值 | `ctx.credentials` | [第 2 章](../part1/02-repo-tour.md) |
| `dsh-credentials-local` | 文件后端 provider（`$DSH_HOME/.env` 叠在进程环境之下） | — | [第 2 章](../part1/02-repo-tour.md) |

## e2b/ — E2B 远程沙箱

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-e2b` | E2B 沙箱生命周期共享层 | — | [第 25 章](../part6/25-sandbox.md) |
| `dsh-fs-e2b` | E2B 文件系统实现 | — | [第 25 章](../part6/25-sandbox.md) |
| `dsh-subprocess-e2b` | E2B 子进程实现 | — | [第 25 章](../part6/25-sandbox.md) |

## examples/ — 示例

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-acp-demo` | ACP 自动化服务器示例：agent 脊柱 + JSONL 持久化 + ACP 传输 | — | [第 37 章](../part9/37-sdk-and-headless.md) |
| `dsh-agent-spine-demo` | 无执行器/无 UI 的默认 agent 脊柱示例 | — | [第 37 章](../part9/37-sdk-and-headless.md) |
| `dsh-sdk-jsonrpc-demo` | 启动外部 Cordis 配置的 stdio JSON-RPC SDK 运行时 bin | — | [第 37 章](../part9/37-sdk-and-headless.md) |

## extensions/ — 动态扩展

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-cordis-host-runner` | 动态包定义注册表、host 半侧沙箱生命周期与 invoke 处理表 | — | [第 33 章](../part8/33-mcp-and-extensions.md) |
| `dsh-cordis-client-runner` | 动态双半插件的浏览器半侧：事件订阅、闭包求值、守卫门面 | — | [第 33 章](../part8/33-mcp-and-extensions.md) |
| `dsh-tool-cordis` | 自指涉 cordis 工具集：检查活跃运行时、挂载与卸载模型编写的插件 | — | [第 33 章](../part8/33-mcp-and-extensions.md) |
| `dsh-client-ui-cordis` | cordis_define 工具卡片与其运行/停止开关 | — | [第 33 章](../part8/33-mcp-and-extensions.md) |

## feedback/ — 反馈

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-command-feedback` | 只写日志的会话反馈生产者与人类斜杠命令 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-message-feedback` | 生命周期绑定的逐消息评分与备注 sidecar | — | [第 35 章](../part9/35-web-server-and-api.md) |

## fs/ — 文件系统

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-fs` | 抽象文件系统接缝：词汇类型、FileSystem 服务与 `fs/*` 策略事件 | `ctx.fs` | [第 23 章](../part6/23-filesystem.md) |
| `dsh-fs-local` | 本地文件系统实现 | — | [第 23 章](../part6/23-filesystem.md) |
| `dsh-fs-sandbox` | 沙箱强制实现：按 per-call 模式围栏写/编辑，读放行 | — | [第 23 章](../part6/23-filesystem.md) |
| `dsh-fs-observation-policy` | 文件上下文策略：observed-state、read-before-edit、版本守卫写入（纯事件门，无服务 API） | — | [第 23 章](../part6/23-filesystem.md) |
| `dsh-tool-fs` | 模型侧文件工具（read、write、edit） | — | [第 23 章](../part6/23-filesystem.md) |
| `dsh-tool-fs-search` | 模型侧发现工具（glob、grep），内置 ripgrep 二进制 | — | [第 23 章](../part6/23-filesystem.md) |
| `dsh-tool-str-replace-editor` | 模型侧 view/create/literal-replace/line-insert 工具 | — | [第 23 章](../part6/23-filesystem.md) |

## goal/ — 会话目标

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-goal` | 事件溯源的同会话 goal 状态与生命周期服务 | `ctx.goals` | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-goal-round-driver` | 竞态围栏的同会话 goal-round driver | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-tool-goal` | 模型侧 goal 工具，带执行时权限检查 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-command-goal` | 人类侧 `/goal` 斜杠命令 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |

## guard/ — 守卫策略

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-repeat-tool-reminder` | 重复工具调用守卫：对同样调用打转时给出建议性提醒 | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |
| `dsh-tool-call-timeout-policy` | 工具调用超时策略：`tools/execute` 包装器按工具布置 deadline，超时返回 TOOL_TIMEOUT | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |

## hooks/ — 外部 hook 兼容

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-hook-protocol` | Claude Code / Codex 共享的 hook 线协议：匹配引擎、stdin/exit-code/stdout 编解码与 `hook/*` 会话事件 | — | [第 31 章](../part8/31-skills-and-hooks.md) |
| `dsh-hooks-claude-code` | 在 dsh 拦截接缝上运行 Claude Code hooks.json 的桥接插件 | — | [第 31 章](../part8/31-skills-and-hooks.md) |
| `dsh-hooks-codex` | 在 dsh 拦截接缝上运行 Codex hooks.json 的桥接插件 | — | [第 31 章](../part8/31-skills-and-hooks.md) |

## host/ — 宿主服务

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-host-apiproxy` | API 网关：ApiProxy 契约、fetch 载体对与 host 侧网关插件 | `ctx.apiProxy` | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-webserver` | Web 路由注册：HTTP/upgrade 路由、index 变换 tap 与静态兜底（不含任何 harness 概念） | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-frontend-static` | Web shell 的 SPA dist 服务器：index-tap 注入、目录穿越拒绝、SPA 兜底 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-plugin-inventory` | 当前 Cordis Loader 插件状态的只读 Remote 投影 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-directory-picker` | 抽象的 workspace 目录选择接缝 | `ctx.directoryPicker` | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-directory-picker-auto` | 自适应选择器：boot 时按宿主情形挂载 native 或 browse 后端 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-directory-picker-browse` | 应用内浏览后端 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-host-directory-picker-native` | 原生 OS 选择器后端 | — | [第 35 章](../part9/35-web-server-and-api.md) |

## identity/ — 身份

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-anonymous-user-id` | 遥测与反馈关联用的共享匿名用户身份 | — | [第 2 章](../part1/02-repo-tour.md) |

## interaction/ — 人机交互

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-commands` | 插件持有的人类命令注册表 | `ctx.commands` | [第 36 章](../part9/36-web-client.md) |
| `dsh-user-approval` | 用户审批接缝：一次性权限决策经 `approval/request` waterfall 分发，默认 fail-closed | `ctx.approval` | [第 20 章](../part5/20-approval-and-presets.md) |
| `dsh-permission-presets` | 用户侧权限预设：一个选择器捆绑 sandbox-mode 与 approval-policy | `ctx.permissionPresets` | [第 20 章](../part5/20-approval-and-presets.md) |
| `dsh-user-questions` | 抽象的"向人提问"接缝 | `ctx.userQuestions` | [第 20 章](../part5/20-approval-and-presets.md) |
| `dsh-tool-ask-user` | 模型侧 `ask_user_question` 工具 | — | [第 20 章](../part5/20-approval-and-presets.md) |

## jobs/ — 后台任务

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-jobs` | 后台任务注册表：共享 id、owner 隔离、轮询、取消与完成监听 | `ctx.jobs` | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-jobs-local` | 进程内实现 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-tool-jobs` | 模型侧任务控制工具（job_output、job_list、job_kill） | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |

## llm/ — 模型层

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-llm` | 供应商中立的 LLM 服务接口 | `ctx.llm` | [第 27 章](../part7/27-llm-vocabulary-and-streaming.md) |
| `dsh-llm-deepseek` | DeepSeek chat-completions 适配器 | — | [第 28 章](../part7/28-adapters.md) |
| `dsh-llm-pi-ai` | pi-ai 后端适配器（`dsh-llm-deepseek` 的设计验证孪生） | — | [第 28 章](../part7/28-adapters.md) |
| `dsh-llm-retry` | 按 provider 路由的 LLM 请求重试策略 | — | [第 29 章](../part7/29-retry-and-token-meter.md) |
| `dsh-token-meter` | 回放感知的 token 计量服务 | `ctx.tokenMeter` | [第 29 章](../part7/29-retry-and-token-meter.md) |

## lsp/ — 语言服务器

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-lsp` | 抽象 LSP 接缝：branded id 键控的 provider 注册表、规范化请求/结果与 LspError 分类 | `ctx.lsp` | [第 26 章](../part6/26-lsp-and-code-runtime.md) |
| `dsh-lsp-stdio` | 通用 stdio 语言服务器 provider：spawn、JSON-RPC 翻译、瞬态打开查询 | — | [第 26 章](../part6/26-lsp-and-code-runtime.md) |
| `dsh-tool-lsp` | 模型侧只读 lsp 工具（定义/引用/实现/hover） | — | [第 26 章](../part6/26-lsp-and-code-runtime.md) |

## mcp/ — MCP

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-mcp-client` | MCP 客户端桥：连接 MCP 服务器并把其工具注册到 `ctx.tools` | — | [第 33 章](../part8/33-mcp-and-extensions.md) |

## plan/ — Plan 模式

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-plan-mode` | 记入日志的 per-agent plan mode：部署引导、直连斜杠命令与用户复核的退出 | — | [第 20 章](../part5/20-approval-and-presets.md) |

## preset/ — Agent 组合

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-agent-presets` | 从 preset cordis.yml 做 per-session agent 组合 | — | [第 7 章](../part2/07-profiles-and-bundles.md) |
| `dsh-persona` | 组合式编写的部署人格 prompt section | — | [第 21 章](../part5/21-system-prompt.md) |

## runtime-diagnostics/ — 运行时诊断

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-invariants` | 包持有的运行时不变量注册服务 | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |

## sandbox/ — 进程沙箱

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-sandbox` | 抽象进程沙箱接缝：同世界约束词汇与 SandboxProvider 契约 | `ctx.sandbox` | [第 25 章](../part6/25-sandbox.md) |
| `dsh-sandbox-local` | 本地后端：bwrap、landlock-run、macOS Seatbelt、Windows ACL——功能探测、fail-closed | — | [第 25 章](../part6/25-sandbox.md) |
| `dsh-sandbox-policy` | per-call 沙箱策略解析器与当前模型上下文（各强制能力族共享） | — | [第 25 章](../part6/25-sandbox.md) |
| `dsh-sandbox-windows-acl` | Windows ACL 写限制后端（受限 token spawn + 能力 SID 写白名单） | — | [第 25 章](../part6/25-sandbox.md) |

## schedule/ — 定时

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-schedule` | 基于会话事件日志的 agent 域持久 after/at/fixed-rate 提醒 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |

## sdk/ — SDK

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-sdk-client` | TypeScript 客户端 SDK：DeepSeekHarness 高层 turns API 与低层 HarnessClient | — | [第 37 章](../part9/37-sdk-and-headless.md) |
| `dsh-sdk-protocol` | SDK 线协议：换行分隔 JSON-RPC stdio 传输与命名的请求/结果/通知类型 | — | [第 37 章](../part9/37-sdk-and-headless.md) |
| `dsh-sdk-jsonrpc-server` | 面向进程外 SDK 客户端的 stdio JSON-RPC 服务器插件 | — | [第 37 章](../part9/37-sdk-and-headless.md) |

## session/ — 会话服务

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-session-persistence` | 抽象持久化接缝 | `ctx.sessionPersistence` | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-persistence-jsonl` | JSONL 持久化后端 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-persistence-sqlite` | SQLite 持久化后端 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-checkpoint-policy` | 模型请求前与工具副作用前的语义持久化检查点 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-projection` | 会话投影接缝：可合并扩展的投影类型表与注册表 | `ctx.sessionProjections` | [第 14 章](../part4/14-derive-messages.md) |
| `dsh-session-projection-cache` | 持久投影缓存：节流写后与冷读阶梯（缓存行 + 持久化尾部回放） | `ctx.sessionProjectionCache` | [第 14 章](../part4/14-derive-messages.md) |
| `dsh-session-stats` | 全日志会话计数与耗时投影 | — | [第 14 章](../part4/14-derive-messages.md) |
| `dsh-session-telemetry` | 遥测接缝：会话事件捕获、投影、脱敏与上报交接 | — | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-session-telemetry-otel` | OpenTelemetry 上报后端 | — | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-session-title` | 日志后备的会话标题服务与 provider 注册表 | `ctx.sessionTitle` | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-session-title-llm` | 标题 provider 共享的 LLM 生成策略 | — | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-session-title-first-prompt-llm` | 基于首条消息的 LLM 标题 provider | — | [第 13 章](../part4/13-session-event-log.md) |
| `dsh-session-title-all-prompts-llm` | 基于全部用户消息的 LLM 标题 provider | — | [第 13 章](../part4/13-session-event-log.md) |

## session-query/ — 会话查询

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-session-query` | 组合式会话查询服务契约：具体读取、trace 与过滤 | `ctx.sessionQuery` | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-query-sqlite` | SQLite FTS5 全文检索后端 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-tool-session-query` | workspace 授权的模型侧历史搜索/trace/事件读取工具 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-session-log-export` | Web 会话日志导出命令与共享下载对话框 | — | [第 17 章](../part4/17-spill-and-storage.md) |

## settings/ — 设置

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-settings` | 抽象用户设置接缝 | `ctx.settings` | [第 2 章](../part1/02-repo-tour.md) |
| `dsh-settings-file` | settings.yaml 文件后端 provider | — | [第 2 章](../part1/02-repo-tour.md) |

## shell/ — Shell 执行

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-shell` | 抽象 bash 执行器接缝（seam 三角的教科书示例） | `ctx.shell` | [第 24 章](../part6/24-execution-world.md) |
| `dsh-bash-local` | 本地子进程实现 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-bash-sandbox` | 沙箱消费实现：每条命令经 `ctx.sandbox` 约束并上报拒绝/强制事实 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-pwsh-local` | 本地 PowerShell 实现 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-pwsh-sandbox` | 沙箱消费的 PowerShell 实现 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-shell-env` | 工具无关的受管 DSH_* shell 环境注册表 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-tool-bash` | 模型侧 bash 工具，可选后台任务与沙箱升级支持 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-tool-bash-persistent` | owner 域持久 Bash 工具，基于 PTY 服务 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-tool-pwsh` | 模型侧 pwsh 工具 | — | [第 24 章](../part6/24-execution-world.md) |

## skill/ — 技能

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-skill` | Agent 技能 provider 注册表 | — | [第 31 章](../part8/31-skills-and-hooks.md) |
| `dsh-skill-filesystem` | 本地文件系统技能 provider | — | [第 31 章](../part8/31-skills-and-hooks.md) |
| `dsh-skill-badge` | 内置 dsh badge 技能 provider | — | [第 31 章](../part8/31-skills-and-hooks.md) |
| `dsh-tool-skill` | 模型侧技能加载工具 | — | [第 31 章](../part8/31-skills-and-hooks.md) |

## spill/ — 溢出存储

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-spill` | 抽象溢出存储接缝：保存超大工具文本并返回取回定位符 | `ctx.spillStore` | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-spill-local` | 本地文件系统实现（会话私有文件） | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-spill-policy` | 工具结果溢出策略：超大纯文本结果替换为预览 + 溢出文件路径（无服务 API） | — | [第 17 章](../part4/17-spill-and-storage.md) |

## storage/ — 通用存储

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-storage` | 存储枢纽：具名后端注册表与挂载的数据形式设施 | `ctx.storage` | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-storage-domain` | 域数据形式：schema 校验、发事件的 KV 域 | `ctx.storage.domain` | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-storage-json` | JSON 文件 KV 后端 | — | [第 17 章](../part4/17-spill-and-storage.md) |
| `dsh-storage-sqlite` | SQLite KV 后端 | — | [第 17 章](../part4/17-spill-and-storage.md) |

## subagent/ — 子代理

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-subagent` | 抽象子代理接缝：委托子代理的具名 provider 注册表 | `ctx.subagents` | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-in-process-driver` | 进程内共享运行 driver（spawn 与 fork 后端复用） | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-spawn-in-process` | 进程内 spawn 后端：在 `ctx.agents` 上跑全新子 agent | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-fork-in-process` | 进程内 fork 后端：以父日志前缀播种子 agent | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-acp` | 进程外 ACP 后端：经 Agent Client Protocol 驱动子进程中的子 agent | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-dsh-sdk` | 进程外 SDK 后端：经 stdio JSON-RPC 驱动子 dsh 运行时 | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-claude-code` | 经官方 Agent SDK 的一次性 Claude Code 子代理 provider | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-subagent-codex` | 经官方 app-server 协议的一次性 Codex 子代理 provider | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-tool-subagent` | 模型侧子代理委托工具 | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-tool-subagent-control` | 全局命名的 send_message / interrupt_agent / list_agents 工具 | — | [第 30 章](../part8/30-subagent.md) |
| `dsh-tool-subagent-report` | 子代理域的 report 工具 | — | [第 30 章](../part8/30-subagent.md) |

## subprocess/ — 子进程

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-subprocess` | 子进程接缝：受管进程组、有界 spill 后备输出与升级 kill | `ctx.subprocess` | [第 24 章](../part6/24-execution-world.md) |
| `dsh-subprocess-local` | 本地实现 | — | [第 24 章](../part6/24-execution-world.md) |

## terminal/ — 持久终端

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-terminal` | 持久 PTY 会话接缝：owner 域 id、后端注册表、交互发送/读取/信号与等待式清理 | `ctx.terminals` | [第 24 章](../part6/24-execution-world.md) |
| `dsh-terminal-bash` | 基于子进程终端原语的持久 shell PTY 后端 | — | [第 24 章](../part6/24-execution-world.md) |
| `dsh-tool-terminal` | 六个模型侧持久 PTY 工具，带 owner 隔离与后台任务集成 | — | [第 24 章](../part6/24-execution-world.md) |

## test-support/ — 测试支撑

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-agent-loop-testkit` | 主循环测试的共享前置挂载 | — | [第 38 章](../part10/38-testing-strategy.md) |
| `dsh-llm-replay` | 回放 LLM 插件：用录制会话 JSONL 重建的模型 chunk 短路 `llm/stream`（免密钥快照测试） | — | [第 38 章](../part10/38-testing-strategy.md) |
| `dsh-llm-mock-server` | 可脚本化的 OpenAI 兼容 HTTP/SSE 故障服务器（恢复测试用） | — | [第 38 章](../part10/38-testing-strategy.md) |
| `dsh-acp-snapshot` | ACP 测试套件：子进程启动器、快照场景 harness 与归一化器 | — | [第 38 章](../part10/38-testing-strategy.md) |
| `dsh-loader-smoke` | 免密钥真实 Loader 示例冒烟测试 harness | — | [第 38 章](../part10/38-testing-strategy.md) |
| `dsh-client-test-runtime` | jsdom slot 测试运行时：真实 Cordis Context + SlotRegistry + 测试替身 | — | [第 38 章](../part10/38-testing-strategy.md) |

## todo/ — Todo

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-tool-todo` | 基于事件溯源会话日志的模型侧 todo_write 工具 | — | [第 18 章](../part5/18-tool-registry.md) |

## typert/ — 类型反射

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-typert-generator` | TypeScript 工程分析器与模型驱动的 Typert 产物生成器 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-typert-loader` | 生成的 Typert 包贡献的 Loader 集成 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-typert-protocol` | 编译器无关的 Remote 元数据与 Typert provider 协议 | — | [第 35 章](../part9/35-web-server-and-api.md) |
| `dsh-typert-registry` | 生成的包反射与 Zod schema 的运行时注册表 | — | [第 35 章](../part9/35-web-server-and-api.md) |

## util/ — 零依赖工具库

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-atomic-write` | 原子文件替换：独占创建随机后缀临时文件 + rename（`writeFileAtomic`） | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |
| `dsh-brand` | 仅类型的 `Branded<B>` 名义类型原语 | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |
| `dsh-home-paths` | 共享的文件系统路径辅助 | — | [第 2 章](../part1/02-repo-tour.md) |
| `dsh-launch-environment` | 不可变的启动环境，记录每个值由哪一层提供 | — | [第 34 章](../part9/34-boot.md) |
| `dsh-native-command` | 无 shell 的 execFile 运行器（宿主原生 OS 集成用） | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |
| `dsh-output-retention` | 有界保留原语：ItemRetainer/TextRetainer 与中性的"保留/省略"提示 | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |
| `dsh-timeout` | 超时/deadline 原语：只做计时与分类，不做终止 | — | [第 39 章](../part10/39-invariants-and-defensive-patterns.md) |

## web/ — Web 访问能力

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-web` | 抽象 Web 访问接缝：search/fetch provider 注册表、注册序无关的选择与 WebError 分类 | `ctx.web` | [第 22 章](../part6/22-seam-triangle.md) |
| `dsh-web-fetch-http` | 匿名公共 HTTP(S) fetch provider | — | [第 22 章](../part6/22-seam-triangle.md) |
| `dsh-web-search-deepseek` | DeepSeek 后端搜索 provider（Anthropic 兼容 API 的原生 web_search） | — | [第 22 章](../part6/22-seam-triangle.md) |
| `dsh-web-search-exa` | Exa 后端搜索 provider | — | [第 22 章](../part6/22-seam-triangle.md) |
| `dsh-web-search-perplexity` | Perplexity 后端搜索 provider | — | [第 22 章](../part6/22-seam-triangle.md) |
| `dsh-tool-web` | 模型侧 web 工具（web_search、web_fetch） | — | [第 22 章](../part6/22-seam-triangle.md) |

## workflow/ — 工作流

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-workflow` | 工作流能力接缝：workflowEngine 服务、运行词汇与 `workflow/*` 事件 | `ctx.workflowEngine` | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-workflow-worker-thread` | worker-thread 工作流引擎：在宿主事件循环之外执行模型编写的编排脚本，`agent()` 调用桥回 `ctx.subagents` | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-tool-workflow` | 模型侧 workflow 工具：运行 JavaScript 编排脚本 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |
| `dsh-tool-ralph` | 基于 workflow 与 subagent 接缝的模型侧 fresh-agent Ralph 循环 | — | [第 32 章](../part8/32-workflow-jobs-goal.md) |

## workspace/ — 工作区

| 包名 | 职责 | 关键 ctx key | 章节 |
|---|---|---|---|
| `dsh-workspace` | 工作区实体注册表：持久 workspace 记录与校验过的会话挂接 | `ctx.workspaceRegistry` | [第 35 章](../part9/35-web-server-and-api.md) |
