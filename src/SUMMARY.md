# Summary

[前言：为什么读 DeepSeek Harness](preface.md)

# 第一部分 全景

- [什么是 Agent Harness](part1/01-what-is-a-harness.md)
- [仓库导览：monorepo 的组织方式](part1/02-repo-tour.md)
- [Everything is a Plugin：整体架构鸟瞰](part1/03-architecture-overview.md)

# 第二部分 Cordis：插件框架基石

- [Context 与 Service：依赖注入的另一种答案](part2/04-context-and-service.md)
- [类型化事件系统：emit / waterfall / parallel / serial](part2/05-typed-events.md)
- [可逆副作用：Effect 与热重载](part2/06-reversible-effects.md)
- [Profiles、Bundles 与配置分层：一棵可补丁的插件树](part2/07-profiles-and-bundles.md)

# 第三部分 Agent 核心循环

- [Turn 与 Step：把对话建模成状态机](part3/08-turn-and-step.md)
- [agent-loop 源码精读：主循环的七百行](part3/09-agent-loop-deep-dive.md)
- [Inbox 与消息注入：输入如何到达模型](part3/10-inbox-and-injection.md)
- [Scope：per-agent 注册的作用域原语](part3/11-scope.md)
- [取消与错误恢复](part3/12-cancellation-and-recovery.md)

# 第四部分 会话即事件日志

- [SessionEvent：append-only 日志与事件溯源](part4/13-session-event-log.md)
- [deriveMessages()：从日志投影模型上下文](part4/14-derive-messages.md)
- ["Model-visible means logged"：一条运行时不变量](part4/15-model-visible-invariant.md)
- [Fork、Resume 与 Compaction](part4/16-fork-resume-compaction.md)
- [Spill 与 Storage：持久化的分层设计](part4/17-spill-and-storage.md)

# 第五部分 工具系统

- [工具注册表与作用域注册](part5/18-tool-registry.md)
- [工具执行流水线：三段 waterfall](part5/19-tool-pipeline.md)
- [审批与权限预设](part5/20-approval-and-presets.md)
- [System Prompt 的组装](part5/21-system-prompt.md)

# 第六部分 能力接缝（Capability Seams）

- [接缝三角：Definition / Provider / Consumer](part6/22-seam-triangle.md)
- [文件系统：fs 抽象与观察策略](part6/23-filesystem.md)
- [Subprocess、Shell 与 Terminal：同一个执行世界](part6/24-execution-world.md)
- [Sandbox：进程约束与远程执行](part6/25-sandbox.md)
- [LSP 与 code-runtime](part6/26-lsp-and-code-runtime.md)

# 第七部分 模型层

- [LLM 词汇表与流式协议](part7/27-llm-vocabulary-and-streaming.md)
- [适配器接缝：llm-deepseek 与 llm-pi-ai](part7/28-adapters.md)
- [重试与 token-meter](part7/29-retry-and-token-meter.md)

# 第八部分 多智能体与扩展生态

- [Subagent：从子代理到跨产品委托](part8/30-subagent.md)
- [Skills 与 Hooks](part8/31-skills-and-hooks.md)
- [Workflow、Jobs、Schedule 与 Goal](part8/32-workflow-jobs-goal.md)
- [MCP 与 Extensions](part8/33-mcp-and-extensions.md)

# 第九部分 宿主与界面

- [Boot：从命令行到插件树](part9/34-boot.md)
- [Web 服务端与 API](part9/35-web-server-and-api.md)
- [Web Client：围绕 session/event 渲染的前端](part9/36-web-client.md)
- [SDK 与 Headless](part9/37-sdk-and-headless.md)

# 第十部分 工程实践与质量文化

- [测试策略：七份 vitest 配置与 test-support](part10/38-testing-strategy.md)
- [Invariants 与防御式编程](part10/39-invariants-and-defensive-patterns.md)
- [文档即工程：生成式目录、i18n 流水线与 postmortem](part10/40-docs-as-engineering.md)
- [Monorepo 工具链：tsconfig 分层、knip、oxlint、lefthook](part10/41-monorepo-toolchain.md)
- [Vendoring：为什么把 Cordis 整个搬进仓库](part10/42-vendoring.md)

# 附录

- [附录 A：术语表](appendix/a-glossary.md)
- [附录 B：包索引](appendix/b-package-index.md)
- [附录 C：事件一览](appendix/c-event-map.md)
