# 第 1 章 什么是 Agent Harness

这一章回答一个看似简单的问题：agent harness 到底是什么，它和 agent framework、agent SDK 有什么区别。我们会列出一个 harness 必须解决的问题清单，然后概览 DeepSeek Harness（下称 dsh）对这份清单的回答方式。读完这一章，你应该能在脑子里建立一张地图：后面四十多章拆解的每个子系统，都对应清单上的某一格。本章代码很少，重在心智模型。

## 问题背景：从一个 while 循环说起

如果你自己动手实现一个 coding agent，朴素做法大概是这样：写一个 while 循环，把用户输入和历史消息发给模型 API，解析返回的 tool call，用一个 `switch` 执行工具，把结果拼回消息数组，直到模型不再调工具。一两百行代码，一个下午就能跑通 demo。

然后坑一个接一个来：

- 换一家模型供应商，流式协议、tool call 格式、reasoning 字段全都不一样，`switch` 里长出第二套解析逻辑；
- 进程一重启会话就没了，想做「继续上次对话」，发现内存里的消息数组和磁盘上存的东西对不上；
- 模型要执行 `rm -rf`，你想在执行前弹个确认框，结果审批逻辑散落在每个工具的实现里；
- 想加沙箱，发现 Bash 工具、文件读写、LSP 各自直接摸本地文件系统，没有一个统一的口子可以拦；
- 想加个 Web UI，发现循环里的中间状态（流式 token、工具进度）根本没有对外暴露的通道；
- 每加一个功能都要改主循环，主循环从两百行长到两千行，谁也不敢动。

这些坑的共性是：**agent 的「循环」本身很小，围绕循环的工程基础设施才是大头**。把这套基础设施做成完整、可运行、可替换的产品外壳，就是 agent harness。

## Harness、Framework、SDK：三个词的分界

这三个词经常混用，但它们回答的是三个不同的问题：

| 形态 | 回答的问题 | 你得到什么 | 典型例子 |
|---|---|---|---|
| **Agent framework** | 「我要开发一个 agent 应用，给我积木」 | 一堆库：chain、memory、tool 抽象，应用由你组装 | LangChain 一类 |
| **Agent SDK** | 「我要把 agent 能力嵌进我的产品」 | 一个客户端：进程内或跨进程调用一个现成的 agent 运行时 | 各家 agent SDK |
| **Agent harness** | 「我要运行、驾驭一个通用 agent」 | 一个完整产品：模型适配、工具、持久化、审批、沙箱、UI 都已接好线，且每一处可换 | Claude Code、dsh |

harness 这个词的本义是「马具/线束」——它不是马（模型），也不是车（你的业务），而是把马和车可靠地连起来的那套带子。一个 harness 拿来就能用（`npx @deepseek-ai/dsh web` 直接起一个 Web UI），同时把每根「带子」都做成可替换件。framework 给你零件让你造车；SDK 给你一辆车的遥控器；harness 给你一辆整车，并且允许你换掉任何一个部件。

dsh 的有趣之处在于它同时是三者：它是一个开箱即用的 harness（`apps/cli` + `apps/web`），也暴露 SDK（`packages/sdk` 的 JSON-RPC 客户端、`python/sdk`），而它的内核 Cordis 插件框架让它天然是一个 framework。这不是刻意的产品定位，而是架构选择的自然结果——这一点我们马上会看到。

## 一个 harness 必须解决的问题清单

把上面的坑系统化，一个 harness 至少要回答六个问题：

1. **模型适配**——不同供应商的消息格式、流式协议、重试语义如何统一？换模型能不能不改循环？
2. **工具执行**——工具如何注册、如何生成给模型看的 schema、执行前后如何插入策略（超时、限流、改写结果）？
3. **会话持久化**——对话如何存储？重启后如何恢复？如何 fork 一个会话、如何在上下文超长时压缩？
4. **权限审批**——哪些操作需要人点头？审批请求如何送达用户、如何回流到正在等待的工具调用？
5. **沙箱**——模型驱动的进程如何被约束在允许的文件和网络范围内？本地执行和远程执行能不能共用一套抽象？
6. **UI 与集成**——终端、浏览器、编辑器、自动化脚本如何观察和驱动同一个 agent？

清单之外还有三个隐性问题，做过的人才知道疼：**扩展性**（第三方如何加能力而不 fork 主循环）、**多智能体**（子 agent 的工具集、权限如何与父隔离）、**取消与错误恢复**（用户按下停止键时，飞行中的模型请求和工具进程如何干净地收拾）。

### 用一条命令串起六问

清单是抽象的，用一个具体场景串一遍就立体了。假设模型决定执行 `rm -rf build/`，在一个合格的 harness 里，这一个动作要依次穿过清单上的每一格：

1. 模型的流式响应里出现一个 tool call——**模型适配**层要把不同供应商各异的流式分片，归一成统一的工具调用表示；
2. harness 在注册表里查到 `bash` 工具，并在真正执行前经过一段可插拔的前置流水线——**工具执行**层在这里做超时、参数校验、策略改写；
3. 前置流水线里的审批策略判断这是个破坏性命令，挂起执行、向用户发起请求——**权限审批**层要保证「等待中」这个状态可被 UI 呈现、可被批准或拒绝、批准后原地续跑；
4. 用户点了同意，命令真正 spawn 时被包上一层进程约束，只能写工作区内的路径——**沙箱**层；
5. 从 tool call 出现、审批发起、到命令输出返回，每一步都作为事件追加进会话日志——**会话持久化**层保证此刻断电重启，对话依然能原样恢复；
6. 终端里的转圈、浏览器里的审批卡片、编辑器插件里的通知，全部由同一条事件流渲染——**UI** 层不需要私有通道。

这条链上任何一格缺失，都会退化成前面那个 while 循环的某个坑。dsh 的完整答案分布在本书各部分：第 3 步的审批流水线在[第 20 章](../part5/20-approval-and-presets.md)，第 4 步的 `ctx.sandbox` 在[第 25 章](../part6/25-sandbox.md)，第 5 步的事件日志是整个第四部分的主题。而 harness 之所以是「产品外壳」而非「组件库」，正因为它对这六格给出的是一套**已经接好线**的默认答案——你替换任何一格，其余五格照常工作。

## dsh 的回答：Everything is a Plugin

dsh 对整份清单只有一个回答：**一切皆插件**。`README.md` 开篇就是这句话，`docs/architecture.md` 把它说得更狠：

> There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads.

没有特权核心。模型适配器是插件，工具注册表是插件，会话日志是插件，**连 agent 主循环本身也是插件**（`packages/core/agent-loop`，它只是 `packages/core/agent` 所定义接口的默认实现，随时可以被整个换掉）。所有插件挂在一个由 Cordis 框架管理的共享 context 上，通过三种机制协作：**服务**（`ctx.tools`、`ctx.llm` 这样的具名能力）、**类型化事件**（`agent/pre-step`、`tools/execute` 这样的扩展点）、**可逆副作用**（每次注册都返回 disposer，插件卸载时自动回滚）。

一个运行中的 dsh，就是一棵在启动时由配置层叠出来的插件树。看一眼每个 profile 的第一层 `dsh-base` bundle 的补丁文件：

```yaml
# packages/bundle/base/cordis.patch.yml
- insert:
    - id: timer
      name: '@deepseek-ai/cordis-plugin-timer'

    - id: hmr
      name: '@deepseek-ai/cordis-plugin-hmr'
      config:
        root: ['.']

    - id: llm
      name: '@deepseek-ai/dsh-llm'

    - id: session
      name: '@deepseek-ai/dsh-session'
    # ...
```

产品不是代码里 `import` 出来的，而是这样一行行「配置行」插出来的。每行有一个 `id`，上层配置（用户的 `cordis.patch.yml`、命令行 `--patch`）可以按 `id` 定点替换任何一行。`dsh --profile web --dump-config` 会打印你机器上实际启动的整棵树——**你看到的每一行都可以被你自己的补丁换掉**。profiles 与 bundles 的层叠细节留给[第 7 章](../part2/07-profiles-and-bundles.md)。

在这个地基上，清单六问各自落到一组插件包上（均可在 `packages/` 下用 `ls` 验证，详见[下一章](02-repo-tour.md)）：

| 问题 | dsh 的回答 | 所在 | 详解章节 |
|---|---|---|---|
| 模型适配 | `ctx.llm` 上的 adapter 接缝 + 统一流式词汇表 | `packages/llm/` | [第 27–29 章](../part7/27-llm-vocabulary-and-streaming.md) |
| 工具执行 | `ctx.tools` 作用域注册表 + 三段 waterfall 流水线 | `packages/core/tools` | [第 18–19 章](../part5/18-tool-registry.md) |
| 会话持久化 | append-only 的 `SessionEvent` 日志，模型上下文由日志投影 | `packages/core/session` + `packages/session/` | [第 13–17 章](../part4/13-session-event-log.md) |
| 权限审批 | approval/interaction 接缝 + permission preset | `packages/interaction/` | [第 20 章](../part5/20-approval-and-presets.md) |
| 沙箱 | `ctx.sandbox` 进程约束接缝，bwrap/Landlock/Seatbelt 后端 | `packages/sandbox/`、`native/landlock-run` | [第 25 章](../part6/25-sandbox.md) |
| UI 与集成 | 全部围绕 `ctx.agents` 驱动、从 `session/event` 渲染 | `packages/host/`、`packages/client/`、`packages/sdk/`、`packages/acp/` | [第 34–37 章](../part9/34-boot.md) |

注意最后一行的措辞：Web UI、ACP 自动化服务器、JSON-RPC SDK 不是三套集成代码，它们是同一句话的三次重复——「驱动 `ctx.agents`，从 `session/event` 渲染」。这正是插件架构的红利：解决一次，处处适用。

三个隐性问题也有各自的原语：扩展性靠文档化的事件扩展点（`docs/architecture.md` 里有一张「Where new behavior goes」总表，[第 3 章](03-architecture-overview.md)会提炼它）；多智能体靠 scope（per-agent 注册作用域，[第 11 章](../part3/11-scope.md)）和 subagent 接缝（[第 30 章](../part8/30-subagent.md)）；取消恢复靠贯穿全循环的 `AbortSignal` 与结构化的 turn 结束原因（[第 12 章](../part3/12-cancellation-and-recovery.md)）。

## 两条贯穿全书的纪律

除了「一切皆插件」，dsh 源码里还有两条反复出现的纪律，先在这里立好路标。

**第一条：能力以「接缝」（seam）为单位存在。** 一个 seam 由三个角色组成——Service Definition（声明接口的抽象 Service）、Service Provider（实现）、Consumer（使用者，通常是模型可见的工具）。`packages/shell` 是官方钦定的教科书案例：`dsh-shell` 定义接口，`dsh-bash-local` / `dsh-bash-sandbox` 是两个 provider，`dsh-tool-bash` 是 Consumer。仓库根 `AGENTS.md` 明确规定「seam 是完整能力，绝不是单个角色」。这就是为什么把文件系统和子进程的 provider 指向一个远程沙箱，Bash、PTY、LSP 会整体跟着搬过去，不需要 fork 任何工具代码。细节见[第 22 章](../part6/22-seam-triangle.md)。

**第二条：Model-visible means logged。** 任何进入模型请求的内容，都必须能从会话日志重建，并且有一条运行时不变量在每次 LLM 请求前实际断言这件事（`packages/core/agent-loop/src/invariant.ts`，[第 3 章](03-architecture-overview.md)会看到它的源码）。这条纪律把「会话持久化」从一个存储功能升格为整个系统的公理：fork、resume、转录、遥测全部从同一条日志推导。细节见[第 15 章](../part4/15-model-visible-invariant.md)。

> 💎 **设计亮点：连主循环都不是特权代码。** 普通做法是把 agent loop 写成应用骨架，插件系统挂在循环身上；dsh 反过来，循环（`dsh-agent-loop`）只是 `ctx.agentLoop` 这个服务的默认 provider，`packages/README.md` 明文规定「扩展插件依赖 Service Definition（`dsh-agent`），绝不依赖具体 provider」。这意味着换掉整个循环实现是一次配置操作而不是一次 fork——「一切皆插件」不是口号，是被依赖规则强制执行的约束。

> 💎 **设计亮点：产品形态 = 配置层叠，且层叠结果可打印。** 普通做法里「headless 版」「Web 版」是代码里的 if 分支或不同入口文件；dsh 里它们是 `dsh-headless` / `dsh-web-app` 两个 bundle 包，各自只是一叠配置行，压在共同的 `dsh-base` 之上。`--dump-config` 把最终的树打印出来，每一行都能被用户按 id 补丁替换。「可扩展性」由此获得一个可验证的定义：你能看到的，就是你能换的。

> 💎 **设计亮点：用一份词汇表钉死概念边界。** `docs/glossary.md` 给每个概念规定唯一的正名：seam 只指三角整体、turn/step/round 三层循环各有精确定义、scope 与 lineage 严格分开。普通项目的术语靠口口相传，三个月后「session」在五个模块里是五种东西；dsh 把术语表当作 API 来维护，本书附录 A 也以它为准。

## 小结与延伸

Harness 不是更大的 framework，而是一个不同的物种：它是完整的 agent 产品外壳，把模型适配、工具执行、持久化、审批、沙箱、UI 全部接好线，同时保证每根线可替换。dsh 的答案是把「可替换」推到极限——没有特权核心，一切（包括循环本身）皆插件，产品形态由配置层叠决定。两条纪律贯穿全书：能力以 seam 三角为单位，模型可见即已记录。下一章我们把仓库结构摸一遍，看看这套理念如何映射到目录布局。

**阅读清单**（相对仓库根）：

- `README.md` — 一分钟了解 dsh 是什么、怎么跑起来
- `docs/architecture.md` — 官方架构总图，本书第 3 章的底图
- `docs/glossary.md` — 术语表：seam、scope、turn/step/round、goal、Ralph
- `AGENTS.md` — 写给 agent 的仓库规则，浓缩了几乎所有工程纪律
- `docs/cordis-primer.md` — Cordis 五个核心概念的速成页（第 4 章前的预习材料）
