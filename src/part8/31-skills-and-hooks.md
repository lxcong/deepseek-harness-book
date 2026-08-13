# 第 31 章 Skills 与 Hooks：两条「外部内容进入会话」的通道

Skills 让用户用一个 markdown 文件给 agent 加一套任务专用指令；Hooks 让用户用一条 shell 命令拦截 agent 的生命周期节点。两者是 harness 对外开放度最高的两个面——内容不由仓库作者控制、随时可变、格式还要兼容别家产品（Claude Code、Codex 的既有生态）。这一章读 `packages/skill/` 的四个包和 `packages/hooks/` 的三个包，看它们如何在不碰 system prompt、不碰 agent-loop 的前提下，把不可信的外部内容安全地送进会话。

## 问题背景

朴素的 skill 实现是把所有技能全文拼进 system prompt：技能一多 context 直接爆炸，而且技能正文变一次就使整个 prompt cache 失效。第二版通常改成"目录 + 按需加载"，但马上遇到一串新问题：技能散在项目目录、用户目录、内置目录，同名冲突谁赢？文件被编辑器改了、被模型自己的 `write` 工具改了，目录何时失效？被禁用的技能还该不该出现在列表里？

朴素的 hook 实现是在工具执行函数里 `if (config.preToolHook) execSync(...)`。它的问题在第三条 hook 出现时爆发：多个 hook 的裁决怎么合并？deny 和 allow 打架谁赢？hook 想给模型塞上下文但不想否决怎么办？更麻烦的是方言——Claude Code 和 Codex 的 hook 配置格式、stdin 约定、退出码语义各不相同，硬编码一种就锁死了生态。

DeepSeek Harness 的做法：skill 走「分层 provider 注册表 + progressive disclosure」，hook 走「中立线协议 + 每方言一个 bridge」，两者的会话入口全部复用第 5 章的 waterfall 事件和[第 10 章](../part3/10-inbox-and-injection.md)的消息注入。

## 源码剖析

### Skill 的数据模型：正文只存在于 Definition

`packages/skill/skill` 是 seam 包（`ctx.skills`），类型分四层：`SkillSummary`（目录条目）→ `SkillCandidate`（provider 报出的候选，多一个 `rank` 和不透明 `locator`）→ `SkillDefinition`（完整定义）→ 运行时注册。关键在字段分布：

```ts
// packages/skill/skill/src/index.ts
/** Invocation-neutral skill metadata returned by `ctx.skills.list()`. */
export interface SkillSummary {
  readonly name: string
  readonly description: string
  readonly whenToUse?: string
  readonly invocation: SkillInvocationPolicy
  readonly source: SkillSource
  readonly provider: string
  readonly resourceBase?: SkillResourceBase
}

/** Complete parsed skill definition, including the body loaded by `ctx.skills.get()`. */
export interface SkillDefinition extends SkillSummary {
  /** Markdown instruction body after any provider-specific metadata removal. */
  readonly content: string
  // ...
}
```

`content` 只出现在 `SkillDefinition` 上。`ctx.skills.list()` 返回 `SkillSummary[]`，`toSummary()` 用显式解构白名单收窄字段——正文和绝对路径在类型上就到不了目录消费者手里。这就是 progressive disclosure 的第一层：**列表永远只有 name + description，正文要用 `get()` 单独换**。第二层藏在 `renderResourceHint` 里：技能带资源目录时，注入的只是"Base directory for this skill: ... Load referenced resources only as needed"一句话，绝不枚举目录内容。

`SkillInvocationPolicy` 有两个开关：`modelInvocable`（frontmatter 的 `disable-model-invocation`）和 `userInvocable`（`user-invocable`）。注册表本身 invocation-neutral——从不过滤，四种组合全保留，由每个消费者在自己的边界执行策略。

### skill-filesystem：六个根、一台 watcher、两条失效路径

`skill-filesystem`（1041 行，整个子系统最大的文件）是文件系统 provider。它扫六类根目录，rank 决定同层同名谁赢（小者胜）：

| rank | source | 目录 |
|---|---|---|
| 100 | `project-dsh` | `<projectRoot>/.dsh/skills` |
| 200 | `project-agents` | `<projectRoot>/.agents/skills` |
| 300 | `custom` | `Config.customSkillDirs` |
| 400 | `user-dsh` | `~/.dsh/skills`（跳过 `.system`） |
| 500 | `user-agents` | `~/.agents/skills` |
| 600 | `bundled` | 内置技能目录 |

projectRoot 是最近含 `.git` 的祖先。扫描不递归：每个根下只认 `<name>/SKILL.md` 目录包和 `<name>.md` 扁平文件两种形态。frontmatter 解析是手写的（找 `---` 分隔、只对中间段跑 `yaml.parse`），而且**任何一步失败都是 `logger.warn` + 跳过该文件**，一个坏 SKILL.md 永远不会毁掉整轮发现（`parseSkillFile`）。frontmatter 里还有一个小的防错设计：旧的 camelCase 键名（如 `disableModelInvocation`）被显式拒绝并指向 kebab-case 正名，而不是静默忽略。

失效有两条互补路径：

```ts
// packages/skill/skill-filesystem/src/index.ts
export function apply(ctx: Context, config: Config = {}): void {
  let provider!: FileSystemSkillProvider
  ctx.skills.registerProvider((control) => {
    provider = new FileSystemSkillProvider(ctx, control, config)
    return provider
  })
  ctx.effect(function* () {
    yield async () => { await provider.dispose() }
  }, 'skill-filesystem watcher')
  ctx.on('fs/observed', (target, _observation, actor) => {
    if (mutationToolName(actor) === undefined) return
    provider.observeHostMutation(target.displayPath)
  })
}
```

外部改动（IDE、git）走 chokidar watcher（`depth: 1`、`awaitWriteFinish` 防半写文件；根目录还不存在时从最近存在的祖先起用 `fs.watchFile` 逐段轮询等它出现）；模型自己用 `edit`/`write` 工具改技能文件则走 `fs/observed` 事件同步失效——[第 23 章](../part6/23-filesystem.md)的 fs 观察接缝在这里收到了回报：watcher 有延迟，而模型改完技能下一步就可能要用，同步路径保证它看到的目录不落后于自己的写入。

还有一个容易做错的细节被显式建模了：watcher 挂不上时，provider 返回 `{ candidates, complete: false }`——技能照常可用，但这次观测"不完整"。注册表对不完整观测**拒绝缓存**，消费者（tool-skill）看到 `!snapshot.complete` 就保留上一次的 last-good 目录不动。"权威性的缺席"和"暂时性的故障"被分成两件事。

### tool-skill：目录、工具、手势——三个入口一种渲染

`tool-skill` 注册模型可见面。第一入口是 `skill` 工具（参数就一个 `name`）；第二入口是会话目录——注意它**不是 system prompt section**，而是一条 `source.kind === 'skill-catalog'` 的持久 user 消息，在 `agent/pre-step` waterfall 里首发或替换：

```ts
// packages/skill/tool-skill/src/index.ts — renderCatalogMessage() 节选
'<system-reminder>',
'A skill is a reusable set of task-specific instructions. The following skills are available in this session:',
// ...
"... This catalog contains summaries only; do not infer or follow a skill's
instructions until it has been loaded.",
```

目录变更判定不比较渲染后的散文，而是对结构化 entries 做 sha256 摘要（`digestCatalogEntries`，每条 entry 先 `JSON.stringify` 再拼接——注释解释了为什么不用分隔符：任何分隔符都是 description 的合法字符，只有引号才能让边界精确）。工具不可见时目录立即清空重发——schema 和调用指引同生共死，靠的是精确的定义身份比较 `ctx.tools.get(skillTool.name, agent) === skillTool`，防止一个恰好也叫 `skill` 的 scoped 影子工具继承这份目录。

第三入口是用户手势：claim 到的 user 消息里出现空白边界的 `/skill-name`（正则特意排除 `/usr/bin` 和 `5/8`），就在同一个 `agent/pre-step` waterfall 里把技能正文作为注入消息追加：

```ts
// packages/skill/tool-skill/src/index.ts
ctx.on('agent/pre-step', async ({ agent, messages, signal }, next): Promise<PreStepDecision> => {
  const decision = await next()
  if (decision.kind === 'reject') return decision
  const names = invokedSkillNames(messages)
  // ...
  for (const name of names) {
    const skill = await ctx.skills.get(name, lookup)
    if (skill === undefined || !isUserInvocable(skill)) continue
    const source: SkillInvocationSource = { kind: 'skill-invocation', name, form: 'instructions' }
    injections.push(createUserMessage({ content: [{ type: 'text', text: renderSkillContent(skill) }], source }))
  }
  if (injections.length === 0) return decision
  return { kind: 'enter', messages: [...decision.messages, ...injections] }
})
```

这条手势路径是 `disable-model-invocation` 技能的**唯一**入口——目录和 `skill` 工具都看不见它们。而工具路径和手势路径共用同一个 `renderSkillContent()`（`packages/skill/skill/src/index.ts`），模型在两条路上看到的都是同一个 `<skill_content>` 包裹结构。两个 pre-step 监听器的注册顺序被当成契约写进注释：手势监听器先注册，waterfall 交给它的是已带目录的消息列表，于是技能正文永远排在目录之后、最贴近模型的下一步动作。

顺带纠正一个望文生义的误解：`skill-badge` 不是什么"prompt 徽章机制"，它是一个 60 行的内置 skill——`dsh-badge`，教模型怎么给 PR 贴 "powered by dsh" 徽章。它的价值在于是 `SkillProvider` 接口的最小样板：`list()` 返回一个写死的 candidate，`get()` 读包内 markdown，插件体只有一行 `ctx.skills.registerProvider(() => provider)`。

### hook-protocol：中立协议，方言归 bridge

`packages/hooks` 的结构是「一个协议库 + 两个方言 bridge」。`hook-protocol` 刻意**不定义 hook 事件枚举**——`SessionStart`、`PreToolUse` 这些名字是方言私有的，住在各 bridge 的 `config.ts` 里。协议库只持有五件事：

1. **`CommandHook`**（`{ command, timeoutSec? }`）——两方言共有的命令 hook 形状；CC 的 `prompt`/`agent`/`http` 型 hook 被 bridge 解析后跳过，只有命令 hook 到达 runner。
2. **`runner.ts`**：通过 `ctx.shell`（[第 24 章](../part6/24-execution-world.md)的执行接缝）跑外部命令，payload 以 JSON 写 stdin，**永不抛**——基础设施故障折叠成"无退出码 + stderr 记录"，turn 照常继续。
3. **`codec.ts`**：退出码 + stdout 的解码。两方言共识:退出码 2 = block、stderr 为理由；退出码 0 且 stdout 以 `{` 开头才尝试 JSON。`hookSpecificOutput.hookEventName` 与触发事件不符时，整块 event-scoped 字段被丢弃，但判别符本身仍写进日志——"记录里应该显示这个畸形块声称了什么"。
4. **`matcher.ts`**：一个 `MatcherMode` 参数装下两种方言——CC 对纯 `[A-Za-z0-9_|]+` 模式走字面量 `|` 分割快路径，其余（及 Codex 全部）按正则；运行期非法正则等于不匹配，配置期则用 `matcherDiagnostic` 产出可读诊断让 bridge 在注册前抛掉整份配置。
5. **`merge.ts`**：多 hook 裁决合并。

合并规则值得全文引用它的核心：

```ts
// packages/hooks/hook-protocol/src/merge.ts
/** Rank a single hook's decision for the deny>ask>allow precedence (higher = stricter). */
function rank(decision: HookOutput['decision']): number {
  switch (decision) {
    case 'deny': case 'block': return 3
    case 'ask': return 2
    case 'approve': case 'allow': return 1
    default: return 0 // no decision
  }
}
```

最严者胜；`reason` 按 rank 分桶，只有解释获胜决策的那些被拼接——一个 allow hook 的理由不会混进 deny 的说明里；第一个 `continue: false` 粘住；`additionalContext` 与 `systemMessage` 按顺序累积、不合并。

`detached.ts`（62 行）解决 emit 型钩子的收尾问题：`SessionStart`/`SubagentStart`/`SubagentStop` 没有人 await，若不追踪，`fiber.dispose()` 返回后仍可能有 hook 进程在跑。`createDetachedRuns()` 用一个 in-flight Set + `drain()` 循环 `Promise.allSettled` 直到观测为空——且契约要求 track 的是"run + 后续 continuation"的完整链条，不只是进程。

每次 hook 执行都往 session log 写成对的 `hook/invoked` / `hook/result` 事件，配套 invariant 在事件提交前校验配对（未配对的 `hook/result` 直接 fail）——延续[第 15 章](../part4/15-model-visible-invariant.md)的防御风格。

### 两个 bridge：外部事件名 → 内部接缝

`hooks-claude-code` 在插件加载时**一次性**读取配置文件（`settings.json` 的 `hooks` 键或裸 hooks.json 皆可），解析失败只 warn 并注册零个 hook。它支持 CC 三十来个事件中的 7 个，每个都映射到一个既有内部接缝：

| CC hook | 内部扩展点 | 模式 | 阻断语义 |
|---|---|---|---|
| `SessionStart` | `agent/session-start` | emit（detached） | 不能阻断，context 走 `agent.inject()` |
| `UserPromptSubmit` | `agent/pre-step` | waterfall | deny → `PreStepDecision.reject` |
| `PreToolUse` | `tools/pre-execute` | waterfall | deny/ask → `PreToolDecision` |
| `PostToolUse` | `tools/post-execute` | waterfall | deny → block + feedback |
| `Stop` | `agent/turn-stopping` | serial | 阻断 → `agent.steer()` 强制续跑 |
| `SubagentStart` / `SubagentStop` | `subagent/start` / `subagent/end` | emit（detached） | 只观测 / 注入 |

以 `PreToolUse` 为例，整个映射 8 行——它就是[第 19 章](../part5/19-tool-pipeline.md)三段 waterfall 里 `tools/pre-execute` 的又一个普通监听器：

```ts
// packages/hooks/hooks-claude-code/src/index.ts
ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
  const turn = lastTurn(exec.agent)
  const merged = await runPoint('PreToolUse', exec.name, preToolPayload(ctx, exec), { /* ... */ })
  if (merged.decision === 'deny') return { kind: 'deny', reason: merged.reason ?? 'blocked by PreToolUse hook' }
  if (merged.decision === 'ask') return { kind: 'ask', ...merged.reason !== undefined ? { reason: merged.reason } : {} }
  return next()
})
```

两个 bridge 都遵守一条 waterfall 纪律——**context 不是否决权**。hook 只产出上下文、没有阻断决策时，必须先 `await next()` 把决定权交给下游，再把自己的 context 折到下游的决策上；即使下游是 `block`，context 也照样挂上去。`hooks-codex` 里的注释说得直白："Context alone is not a veto: DELEGATE so a later pre-step listener can still reject/rewrite, then fold our context onto its decision."

`hooks-codex` 的差异全是显式的：matcher 恒为正则（无字面量快路径）、stdin 无尾换行（`trailingNewline: false`）、`PreToolUse` 只认 deny 不认 allow/ask、纯文本 stdout 可选地当 additionalContext、payload 带 `model`/`permission_mode`/`turn_id`。其中一处见功力：Codex payload 的 `tool_input` 压成它习惯的 `{ command }` 形状，但 `tool_name` 保留真实工具名——注释解释：写死常量会让"matcher 测试的名字"与"payload 声称的名字"不一致，用户配置的工具 matcher 将永远打不中。

## 设计亮点

> 💎 **设计亮点：用字段分布实现 progressive disclosure**。"目录只给摘要、正文按需加载"通常靠代码纪律维持，这里靠类型：`content` 只存在于 `SkillDefinition`，`list()` 的返回类型是 `SkillSummary[]`，`toSummary()` 显式解构白名单——正文"忘了删"在编译期就不可能。附带的红利写在 `docs/subsystems/skills.md`：只改技能正文不产生任何目录消息、不重写历史 tool result，因为注册表根本不缓存完整定义，目录摘要只吃 name + description。

> 💎 **设计亮点：不完整观测是一等公民**。`SkillProviderObservation` 允许 provider 声明"这批 candidate 可用，但这次没扫全"；注册表对不完整结果拒绝入缓存，消费者保留 last-good 目录。watcher 起不来时技能照常可读，只是观测不权威。普通写法要么在故障时抛异常（技能全部消失），要么静默返回部分结果并污染缓存——这里把两种坏结局都堵死了。

> 💎 **设计亮点：库持有参考默认值，配置权归 bridge**。`hook-protocol` 定义了 `DEFAULT_HOOK_TIMEOUT_MS`、`DEFAULT_STDERR_SUMMARY_MAX_CHARS`，但库内部一处都不用——runner 和 events 只吃调用方显式传入的参数。两个 bridge 引用同一常量所以不会漂移，但"超时该是多少"的决定权始终在持有用户配置的 bridge 手里。加上 `MatcherMode` 一个参数装下两种方言语义，dialect 差异永远是显式参数而非复制粘贴的分支。

> 💎 **设计亮点：最严者胜 + 分桶理由**。`mergeHookOutputs` 的 deny > ask > allow 排序是显然的，不显然的是 reason 按 rank 分桶：最终决策是 deny 时，只有 deny hook 们的理由被呈现，allow hook 的"我觉得没问题因为……"不会混进拒绝说明误导模型。一个 Map<rank, string[]> 就消灭了一类"合并后语义自相矛盾"的 bug。

## 小结与延伸

Skills 和 Hooks 是 harness 生态兼容性的试金石：前者兼容 `.agents/skills` 的通用技能格式，后者直接消费 Claude Code 和 Codex 的既有 hook 配置。两者的实现路数一致——中立的核心（invocation-neutral 注册表 / dialect-free 协议库）+ 边界上的策略执行（消费者过滤 / bridge 映射），而所有会话入口都是既有接缝的普通监听器：目录和技能正文走 `agent/pre-step` waterfall 注入，hook 裁决走 `tools/pre-execute`、`agent/turn-stopping` 等扩展点。system prompt 自始至终没有被碰过——技能目录是持久的 user 消息，这让它能随会话 fork/resume 一起被 [第 14 章](../part4/14-derive-messages.md)的投影自然恢复。

延伸阅读：

- `docs/subsystems/skills.md` —— skill 子系统官方文档（注意：hooks 没有对应的 subsystems 文档，权威文字在各包 README）
- `packages/hooks/hooks-claude-code/README.md` —— 完整映射表与 23 个未支持事件的清单
- `packages/skill/skill/src/index.ts` —— 分层注册表、scope 链合并与缓存键设计
- `.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md`、`2026-06-30-hook-bridges.md` —— hook 扩展点的设计笔记
- [第 32 章](32-workflow-jobs-goal.md) —— 同样只靠接缝接入的六个子系统
