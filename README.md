# DeepSeek Harness 源码解读

> 从 Cordis 插件树到 Agent 主循环：一次关于工程优雅的源码之旅

本书面向正在学习 Agent Harness 工程实现的工程师，逐层解读 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）的源码：它如何用「everything is a plugin」的架构组织一个 50+ 包的 monorepo，如何把会话建模成 append-only 事件日志，如何用「能力接缝」让一次 provider 替换搬迁整个执行世界。

我们关心的不是「这个项目能做什么」，而是「它为什么这样写、换一种普通写法会失去什么」。

## 阅读方式

- 在线阅读：用 [mdBook](https://rust-lang.github.io/mdBook/) 构建——`mdbook serve` 后访问 http://localhost:3000
- 直接阅读：从 [src/SUMMARY.md](src/SUMMARY.md) 进入各章 markdown
- 建议搭配源码：`git clone https://github.com/deepseek-ai/deepseek-harness` 并对照书中标注的文件路径阅读

## 全书结构

| 部分 | 主题 | 关键词 |
|---|---|---|
| 一 | 全景 | agent harness 概念、monorepo 导览、架构鸟瞰 |
| 二 | Cordis 插件框架 | Context/Service、类型化事件、可逆 effect、profiles/bundles |
| 三 | Agent 核心循环 | turn/step、agent-loop、inbox、scope、取消恢复 |
| 四 | 会话即事件日志 | SessionEvent、deriveMessages、不变量、fork/compaction |
| 五 | 工具系统 | 注册表、三段 waterfall 流水线、审批、system prompt |
| 六 | 能力接缝 | seam 三角、fs、执行世界、sandbox、LSP |
| 七 | 模型层 | 流式协议、适配器、重试、token 计量 |
| 八 | 多智能体与扩展 | subagent、skills/hooks、workflow/jobs/goal、MCP |
| 九 | 宿主与界面 | boot、web server、client、SDK/headless |
| 十 | 工程实践 | 测试策略、invariants、文档即工程、工具链、vendoring |

## 对应源码版本

本书基于 `deepseek-ai/deepseek-harness` 的 developer preview 阶段源码写作（2026-08）。上游仍在快速迭代并会有破坏性变更，个别细节可能与最新 `main` 不一致；每章末尾的「延伸阅读」列出了对应的源码路径，以便对照当前版本。

## 协议

本书以 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 发布；引用的 DeepSeek Harness 源码遵循其 [MIT 协议](https://github.com/deepseek-ai/deepseek-harness/blob/main/LICENSE)。
