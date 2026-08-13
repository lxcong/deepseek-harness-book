# 前言：为什么读 DeepSeek Harness

## 这是一个什么项目

DeepSeek Harness（命令行名 `dsh`）是 DeepSeek AI 开源的 agent harness——也就是包住大模型、让它能持续工作的那层"驾驭装置"：会话、工具、审批、沙箱、子代理、Web 界面，全部在内。

仓库 README 只用了一句话概括它的架构：**everything is a plugin**，底座是插件框架 [Cordis](https://github.com/cordiverse/cordis)。

这句话不是宣传语，而是字面事实。翻开 `docs/architecture.md` 你会看到：模型适配器是插件，工具注册表是插件，会话日志是插件，连 agent 主循环本身也是插件。没有一个"特权核心"供你打补丁——扩展 dsh 的方式是把你的插件挂到其他插件旁边，而所有注册都是可逆的 effect，插件卸载时自动回退。

一个运行中的 `dsh` 就是启动时由有序图层组合出来的一棵插件树：每个 bundle 按序铺底，profile 的 `cordis.patch.yml` 盖上一层，home 级 patch 再盖一层，最后是 `--patch` overlay。`dsh --profile web --dump-config` 能把这棵树原样打印出来，且其中任何一行都可以被你的 patch 替换。

项目目前处于 developer preview 阶段，迭代很快、明确声明会有破坏性变更。但这恰恰是读源码的好时机：架构骨架已经成型，代码量还没膨胀到读不动。

## 为什么值得工程师精读它的源码

市面上的 agent 框架不少，值得逐行精读的不多。这个仓库有四个特质，让"读它"本身成为一次系统性的工程训练。

**第一，everything-is-a-plugin 被贯彻到了底。**

大多数框架说自己"支持插件"，意思是核心之外留了几个钩子。dsh 反过来：`packages/` 下两百多个包，从 `core/agent-loop` 到 `util/timeout`，全部以同一种方式（服务、类型化事件、可逆 effect）挂进同一棵 Cordis 树。你想换掉 bash 执行器、换掉持久化后端、甚至换掉整个 agent 循环，手段是同一个——patch 掉那一行配置。这种一致性带来的架构纪律，值得在源码里逐处观察。

**第二，会话是一条事件溯源日志。**

dsh 的会话不是一个可变的消息数组，而是 append-only 的 `SessionEvent` 日志；模型看到的上下文由 `deriveMessages()` 从日志投影出来，fork、resume、compaction、转录、遥测全部派生自同一条流。仓库里还有一条运行时不变量把这个设计钉死："model-visible means logged"——凡是进入模型请求的内容，必须能从日志重建。这是事件溯源在真实产品里的一次完整落地。

**第三，能力接缝（capability seam）。**

dsh 把每一种能力拆成三角：Service Definition（接口）、Service Provider（实现）、Consumer（通常是面向模型的工具）。文件系统和子进程 provider 共享同一个"执行世界"，于是把它们指向远程沙箱，Bash、PTY、LSP 就整体跟着搬走，不需要 fork 任何一个 provider。`packages/shell` 是教科书式的例子：`dsh-shell` 定义、`dsh-bash-local` / `dsh-bash-sandbox` 实现、`dsh-tool-bash` 消费。

**第四，工程化的文档与质量文化。**

`docs/event-producer-consumer.md` 不是手写的——它由脚本从 TypeScript Program 里解析出每个事件的生产者与消费者自动生成；事件的 dispatch mode 用 `@mode` 标签声明并被生成式目录校验；文档全部配有 i18n 流水线；仓库甚至保留了 postmortem 目录。读这个仓库，顺带能学到一套"文档即工程"的做法。

## 本书写给谁

本书的目标读者是正在学习 agent harness 工程实现的工程师。更具体地说，你大概处在下面某种处境：

- 你在做自己的 agent 产品，写到第三个月发现工具注册、审批、会话持久化、子代理这些东西越堆越乱，想看看一个把架构想清楚了的项目是怎么组织它们的；
- 你日常使用 Claude Code、Codex 这类编码 agent，好奇它们那层"harness"内部到底长什么样，而 dsh 恰好是一个结构清晰、可以整只解剖的开源标本；
- 你对插件架构、事件溯源、依赖注入这些设计手法感兴趣，想找一个把它们同时用到位的真实代码库，而不是玩具示例。

前置要求不高：能读 TypeScript，理解 Promise 与 async/await，对 LLM API 的基本形态（消息数组进、流式 token 出、tool call）有概念即可。**不需要**事先了解 Cordis——第二部分会从零讲起；也不需要用过 dsh 本身。

反过来，本书**不是**什么：它不是 dsh 的使用手册（那是仓库 `docs/user/` 的职责），不教你写 prompt，也不比较各家 agent 框架的优劣。全书只做一件事：把一个真实 harness 的源码摊开，讲清楚每个子系统为什么这样设计。

## 本书结构

全书十个部分加三个附录，按"由外到内、再由内到外"的顺序展开。

- **第一部分 全景**——先回答什么是 agent harness，再带你走一遍 monorepo 的目录组织，最后鸟瞰 everything-is-a-plugin 的整体架构。
- **第二部分 Cordis：插件框架基石**——精读 Context 与 Service 的依赖注入、emit / waterfall / parallel / serial 四种类型化事件分发、可逆 effect 与热重载、profile / bundle / patch 的配置分层。
- **第三部分 Agent 核心循环**——把对话建模成 turn / step 状态机，逐段精读 `core/agent-loop`，看输入如何经 inbox 到达模型、scope 如何实现 per-agent 注册、取消与错误恢复如何兜底。
- **第四部分 会话即事件日志**——append-only 的 `SessionEvent` 日志、`deriveMessages()` 投影、"model-visible means logged" 不变量、fork / resume / compaction，以及 spill 与 storage 的持久化分层。
- **第五部分 工具系统**——工具注册表与作用域注册、pre-execute / execute / post-execute 三段 waterfall 流水线、审批与权限预设、system prompt 的组装。
- **第六部分 能力接缝**——Definition / Provider / Consumer 三角，然后逐一走查 fs、subprocess / shell / terminal 组成的执行世界、sandbox 与远程执行、LSP 与 code-runtime。
- **第七部分 模型层**——LLM 词汇表与流式协议、`llm-deepseek` 与它的设计验证孪生 `llm-pi-ai`、重试策略与 token-meter。
- **第八部分 多智能体与扩展生态**——subagent 的多种 provider（从进程内子代理到委托给另一个产品）、skills 与 hooks、workflow / jobs / schedule / goal、MCP 与动态扩展。
- **第九部分 宿主与界面**——从命令行 boot 出插件树、Web 服务端与 API、围绕 session/event 渲染的 Web Client、SDK 与 headless 运行。
- **第十部分 工程实践与质量文化**——测试策略、runtime invariants、生成式文档与 i18n 流水线、monorepo 工具链，以及"为什么把 Cordis 整个 vendor 进仓库"。

三个附录是随手可查的地图：

- [附录 A：术语表](appendix/a-glossary.md)——按主题分组的全书术语，每条链接到首次讲解它的章节。
- [附录 B：包索引](appendix/b-package-index.md)——`packages/` 下全部子包的职责、ctx key 与对应章节。
- [附录 C：事件一览](appendix/c-event-map.md)——按三大事件域分组的完整事件表：dispatch mode、生产者、消费者、用途。

## 如何搭配源码阅读本书

先说清楚本书的立场：**源码是正文，本书是导读**。每一章的代码摘录都注明了相对仓库根的文件路径，我们强烈建议你在本地克隆仓库、把书和编辑器并排打开：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install && pnpm run build
pnpm dsh web
```

几条使用建议：

1. **顺读第一、二部分，之后按需跳读。** Cordis 的 Context / Service / 事件 / effect 四个概念是全书的公共前提；掌握之后，第三到第九部分各自相对独立，可以从你最关心的子系统切入。做 agent 产品的读者可以先走三、四、五部分（循环—日志—工具）这条主线；做基础设施的读者不妨直接进第六部分看接缝。
2. **善用仓库自带的文档做地图，但以源码为准。** `docs/architecture.md` 是最好的起点，`docs/subsystems/*.md` 和各包 README 覆盖了每个子系统。本书所有结论都回到 `src/*.ts` 求证过，遇到分歧时请相信源码——它比文档和本书都更新。
3. **跑起来再读。** `dsh --profile web --dump-config` 打印你机器上实际 boot 的插件树，对着它读第二部分的配置分层会具体得多；读第四部分时，打开一个会话的 JSONL 持久化文件对照事件流，比任何图都直观。
4. **留意 `> 💎 设计亮点` 引用块。** 每章会标出 2-4 处工程上的优雅之处，并说明"换一种普通写法会怎样"。这些是我们认为最值得带走、迁移到你自己系统里的东西。

## 体例约定

- 技术术语与代码标识符保留英文原文（waterfall、`ctx.effect()`、seam），首次出现时给出中文解释；全书术语与仓库 `docs/glossary.md` 保持一致——一个概念只用一个规范说法。
- 代码摘录注明相对仓库根的路径（如 `packages/core/agent-loop/src/index.ts`），为篇幅删节的部分用 `// ...` 标注。
- 章节间以相对链接互引；提到"仓库文档"时指 `docs/` 目录下的文件。

最后一句提醒：dsh 处在快速迭代期，本书基于写作时点的仓库快照，个别文件路径与行号可能随上游演进而漂移。好在架构骨架——插件树、事件日志、能力接缝——是这个项目最稳定的部分，这也正是本书把笔墨花在骨架而非枝节上的原因。

祝阅读愉快。读完之后，希望你带走的不只是"dsh 是怎么写的"，而是一套可以复用的判断力：下一次你自己设计 agent 系统时，知道哪里该留接缝、哪里该记日志、哪里该让类型系统替你守夜。
