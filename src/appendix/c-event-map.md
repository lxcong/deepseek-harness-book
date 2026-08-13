# 附录 C：事件一览

本表以仓库生成式文档 `docs/event-producer-consumer.md` 为底本（由 `scripts/gen-doc-graphs.ts` 从 TypeScript Program 解析生成），整理为按事件域分组的中文速查表，**保留全部 56 个 harness 事件**。三大事件域的划分依据 `docs/architecture.md`：

- **Session 事件**：追加进日志并经 `session/event` 广播的持久事实；
- **Agent 事件**：携带活跃 `Agent` 的运行中扩展点（inbox、step、status、request 等）；
- **Capability 事件**：不 import 主循环、直接把策略与适配器挂到某个接缝上的事件。

dispatch mode 语义见[第 5 章](../part2/05-typed-events.md)：`emit` 广播不等待；`waterfall` 是 around-middleware，listener 须调用 `next()` 委托、不调用则短路；`parallel` 并行且整体等待；`serial` 顺序执行且被等待。生产者/消费者均为 `packages/` 下的包名（`apiproxy`、`server`、`loader`、`gateway`、`webserver`、`timeout-policy` 为矩阵中的内联标识）。

## Session 事件域

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `session/created` | `emit` | `session` | `apiproxy`、`compaction`、`goal`、`hook-protocol`、`llm-retry`、`permission-presets`、`plan-mode`、`schedule`、`server`、`session`、`session-persistence`、`session-telemetry`、`time-context`、`tool-workflow`、`tools`、`user-approval` | 新会话创建后的广播；各持久域与策略插件在此挂接会话级状态 |
| `session/disposed` | `emit` | `session` | `agent-loop`、`apiproxy`、`session-persistence`、`session-projection-cache`、`session-telemetry`、`session-title` | 会话对象销毁；消费者释放缓存、句柄与后台工作 |
| `session/event` | `emit` | `session` | `acp`、`agent-instructions`、`agent-loop`、`agent-presets`、`apiproxy`、`compaction`、`compaction-basic`、`goal`、`goal-round-driver`、`hook-protocol`、`loader-smoke`、`server`、`session`、`session-persistence`、`session-projection`、`session-projection-cache`、`session-telemetry`、`session-telemetry-otel`、`session-title`、`token-meter`、`tool-workflow`、`tools`、`user-approval` | 每条 `SessionEvent` 追加后的总线广播；持久化、投影、遥测、标题、UI 推流全部由此驱动（全仓库消费者最多的事件） |
| `session/flush` | `parallel` | `session` | `session-persistence`、`session-telemetry` | 并行等待所有持久化消费者落盘，保证关键节点前日志已写穿 |
| `session-telemetry/record` | `waterfall` | `session-telemetry` | —（当前无 listener） | 遥测记录上报前的改写/脱敏扩展点 |

## Agent 事件域

### agent 生命周期与循环

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `agent-loop/config-start-failed` | `emit` | `agent-loop` | — | 按配置自动开跑失败时的诊断信号 |
| `agent/created` | `emit` | `agent` | `agent-presets`、`goal-round-driver`、`schedule` | 新活跃 agent 进入注册表；preset 组合与调度器在此挂接 |
| `agent/disposed` | `emit` | `agent` | `agent-loop`、`goal-round-driver`、`subagent` | agent 被销毁；驱动方清理续跑与委托状态 |
| `agent/error` | `emit` | `agent-loop` | `acp`、`apiproxy`、`goal-round-driver`、`session-telemetry` | 循环中浮出的错误对外广播（UI 呈现、遥测记录、goal 策略响应） |
| `agent/status` | `emit` | `agent-loop` | `agent`、`apiproxy`、`compaction-basic`、`goal-round-driver`、`schedule`、`server` | agent 状态机变化（空闲/运行等）；UI 状态点与继续策略的依据 |
| `agent/session-start` | `emit` | `agent-loop` | `goal`、`goal-round-driver`、`hooks-claude-code`、`hooks-codex` | 会话首次由该 agent 驱动时触发（对应外部 hook 的 session-start 时机） |
| `agent/pre-step` | `waterfall` | `agent-loop` | `agent-instructions`、`compaction-basic`、`goal-round-driver`、`hooks-claude-code`、`hooks-codex`、`plan-mode`、`repeat-tool-reminder`、`session-checkpoint-policy`、`subagent-in-process-driver`、`time-context`、`tmux-context`、`tool-cordis`、`tool-skill` | 决定模型看到什么：listener 可改写被 claim 的消息或 reject；上下文注入、压缩、plan 模式等策略的主入口（listener 最多的 waterfall） |
| `agent/request` | `waterfall` | `agent-loop` | `agent` | 模型请求发出前的最后包装点 |
| `agent/request-error` | `waterfall` | `agent-loop` | `compaction-basic`、`llm-retry` | 模型请求失败的恢复链：重试或触发压缩后再试 |
| `agent/turn-stopping` | `serial` | `agent-loop` | `hooks-claude-code`、`hooks-codex` | turn 收尾前的顺序钩子（无 `next()`）；外部 stop hook 可在此要求继续 |

### inbox

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `agent/inbox/inserted` | `emit` | `agent-loop` | `goal-round-driver` | 消息进入 inbox |
| `agent/inbox/claimed` | `emit` | `agent-loop` | `acp`、`goal-round-driver`、`subagent`、`tool-jobs` | inbox 消息被 claim 进入当前 turn |
| `agent/inbox/discarded` | `emit` | `agent-loop` | `goal-round-driver`、`subagent` | inbox 消息被丢弃（未被任何 turn 消费） |

### preset、goal、subagent 与 workflow

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `agent-preset/selected` | `emit` | `agent-presets` | `apiproxy` | 某会话选定了 agent preset |
| `goal/changed` | `emit` | `goal` | `goal-round-driver` | goal 状态（phase/revision）变化；round driver 据此决定是否续跑 |
| `subagent/start` | `emit` | `subagent` | `hooks-claude-code`、`subagent` | 一次子代理运行开始 |
| `subagent/end` | `emit` | `subagent` | `hooks-claude-code`、`server`、`subagent` | 一次子代理运行结束 |
| `subagent/provider-added` | `emit` | `subagent` | `subagent`、`tool-subagent` | 子代理 provider 注册；委托工具随之更新可用 provider 列表 |
| `subagent/provider-removed` | `emit` | `subagent` | `subagent`、`tool-subagent` | 子代理 provider 注销 |
| `workflow/start` | `emit` | `workflow` | `workflow` | 一次 workflow 运行开始 |
| `workflow/phase` | `emit` | `workflow` | — | workflow 运行进入新阶段 |
| `workflow/log` | `emit` | `workflow` | — | workflow 脚本的日志输出 |
| `workflow/agent-start` | `emit` | `workflow` | `tool-workflow`、`workflow` | workflow 内一个成员 agent 启动 |
| `workflow/agent-end` | `emit` | `workflow` | `tool-workflow`、`workflow` | workflow 内一个成员 agent 结束 |
| `workflow/end` | `emit` | `workflow` | `workflow` | 一次 workflow 运行结束 |

## Capability 事件域

### 工具（`tools/*`）

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `tools/pre-execute` | `waterfall` | `tools` | `hooks-claude-code`、`hooks-codex`、`tool-jobs` | 工具执行前的准入与改写（外部 pre-tool hook、后台任务化在此介入） |
| `tools/execute` | `waterfall` | `tools` | `session-checkpoint-policy`、`timeout-policy` | 包裹工具执行本体：检查点落盘、超时策略在此包一层 |
| `tools/post-execute` | `waterfall` | `tools` | `hooks-claude-code`、`hooks-codex`、`repeat-tool-reminder`、`spill-policy`、`tool-fs-search` | 结果加工：外部 post-tool hook、重复调用提醒、超大结果 spill |
| `tools/result` | `emit` | `tools` | `agent-instructions`、`subagent-in-process-driver` | 工具结果产出后的观察广播 |
| `tools/change` | `emit` | `tools` | — | 工具注册表自身变化（registry-subject，不做 scope 过滤） |
| `tools/code-dispatch-log` | `waterfall` | `tools` | `spill-policy` | Code Mode 分发日志的改写点（同样受 spill 策略约束） |

### 文件系统（`fs/*`）

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `fs/write-intent` | `waterfall` | `tool-fs`、`tool-str-replace-editor` | `fs-observation-policy` | 写文件意图的策略门：可在真正落盘前拒绝或改写 |
| `fs/edit-intent` | `waterfall` | `tool-fs`、`tool-str-replace-editor` | `fs-observation-policy` | 编辑意图的策略门（read-before-edit、版本守卫在此实施） |
| `fs/observed` | `emit` | `tool-fs`、`tool-str-replace-editor` | `fs-observation-policy`、`skill-filesystem` | 文件被读取/观察的事实广播，维护 observed-state |

### 模型与提示（`llm/*`、`system-prompt/*`）

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `llm/stream` | `waterfall` | `llm` | `agent-loop`、`llm`、`llm-replay`、`session-checkpoint-policy`、`session-title` | 模型流式请求的分发链；`llm-replay` 在测试中于此短路真实请求 |
| `llm/adapters-updated` | `emit` | `llm` | `apiproxy`、`llm` | 适配器注册表变化（模型列表更新） |
| `system-prompt/assemble` | `waterfall` | `system-prompt` | `agent`、`agent-presets`、`system-prompt` | 系统提示组装链：按 scope 合并各插件注册的 prompt section |
| `system-prompt/change` | `emit` | `system-prompt` | — | prompt section 注册表变化 |

### 交互与审批

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `approval/request` | `waterfall` | `user-approval` | `acp`、`apiproxy` | 一次性权限决策分发给组合的 answerer；无人应答则 fail-closed |
| `commands/change` | `emit` | `commands` | `apiproxy` | 人类命令注册表变化（UI 命令目录刷新） |

### 配置、凭证与存储

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `settings/updated` | `emit` | `settings` | `settings` | 某个设置值更新 |
| `settings/document-updated` | `emit` | `settings` | `apiproxy` | 设置文档整体更新（推送给客户端） |
| `credentials/updated` | `emit` | `credentials` | `apiproxy`、`credentials` | 凭证变化广播（值不出 provider，只广播事实） |
| `domain/changed` | `emit` | `storage-domain` | `apiproxy`、`storage-domain`、`workspace` | 域数据形式（KV domain）内容变化 |
| `skills/change` | `emit` | `skill` | — | 技能 provider 注册表变化 |

### 动态扩展（`cordis/*`，host ↔ client 双半插件）

| 事件名 | mode | 生产者 | 消费者 | 用途 |
|---|---|---|---|---|
| `cordis/request-run` | `emit` | `cordis-host-runner` | `apiproxy` | 请求运行一个模型挂载的动态包 |
| `cordis/request-run-resolved` | `emit` | `cordis-host-runner` | `apiproxy` | 上述运行请求已解决 |
| `cordis/dynamic-package` | `emit` | `cordis-host-runner` | `apiproxy` | 动态包定义注册（同步到客户端半侧） |
| `cordis/dynamic-retract` | `emit` | `cordis-host-runner` | `apiproxy` | 动态包定义撤回 |
| `cordis/inspect-query` | `emit` | `cordis-host-runner` | `apiproxy` | 对活跃运行时的检查查询 |
| `cordis/inspect-query-resolved` | `emit` | `cordis-host-runner` | `apiproxy` | 检查查询已解决 |

## 附：包源码中出现的非 harness / 未声明事件串

这些是 Cordis 框架自身的内部事件（不属于上述三域），生成器把监听它们的包也一并列出：

| 事件串 | 监听者 | 说明 |
|---|---|---|
| `internal/dispatch` | `commands`、`compaction`、`fs`、`goal`、`goal-round-driver`、`hook-protocol`、`llm-retry`、`permission-presets`、`plan-mode`、`sandbox-policy`、`schedule`、`scope`、`session`、`session-title`、`subagent`、`terminal-bash`、`time-context`、`tool-todo`、`tool-workflow`、`tools`、`user-approval`、`workflow` | Cordis 事件分发的内部钩子；`core/scope` 正是借它实现 scope 过滤分发 |
| `internal/plugin` | `loader`、`lsp-stdio`、`webserver` | 插件挂载/卸载的框架内部事件 |
| `internal/service` | `agent-presets`、`gateway` | 服务上线/下线的框架内部事件 |
| `internal/status` | `agent` | fiber 状态变化的框架内部事件 |

> 提醒：本表随上游演进会过期。权威版本始终是仓库里再生成的 `docs/event-producer-consumer.md`（`pnpm run gen-doc-graphs`）。
