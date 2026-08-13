# 第 23 章 文件系统：fs 抽象与观察策略

「读文件、写文件」看起来是 harness 里最没什么可讲的部分，但 DeepSeek Harness 的文件系统接缝恰恰是全仓库设计密度最高的地方之一：一个不透明身份 + 版本令牌的 provider 合同（`ctx.fs`），一个继承本地实现只加围栏的沙箱 provider（`fs-sandbox`），以及一个**完全用事件实现、不注册任何服务**的策略插件（`fs-observation-policy`）——「写之前必须先读」这条 agent 编码产品的经典规则，在这里既不写在工具里，也不写在 provider 里。这一章按 Definition → Provider → 策略 → Consumer 的顺序解剖这条接缝，看「read-before-write」如何被拆解成三个可独立卸载的正交部件。

## 问题背景

朴素实现:工具里直接 `fs.readFile` / `fs.writeFile`，再在工具代码里维护一个 `readFiles: Set<string>` 来实现「没读过不许改」。踩过的人都知道坑在哪：

- **路径就是身份**：`./a.ts`、`/repo/a.ts`、symlink 各算一条记录，guard 形同虚设；换成远程文件系统后「路径」根本不是稳定概念。
- **过期写入**：模型 5 分钟前读的文件，用户已经改过了，工具照样整段覆盖。用 mtime 比较？读-比-写三步之间没有原子性，还是有竞态窗口。
- **策略和工具焊死**：想在某个部署里放开「必须先读」的限制（比如批处理管道），只能 fork 工具代码。
- **沙箱重写一切**：要限制模型只能写 workspace 内，最直觉的做法是给每个工具加路径检查——检查逻辑散落各处，且检查的路径和实际写的路径之间又有 TOCTOU。

DeepSeek Harness 把这四个问题分派给了四个不同的部件。

## 源码剖析

### Definition：不透明身份与版本令牌

`ctx.fs` 的所有操作围绕两个 **branded 不透明类型**：

```ts
// packages/fs/fs/src/types.ts
export type FsTargetKey = Branded<'FsTargetKey'>
export type FsVersion = Branded<'FsVersion'>

export interface FsTarget {
  /** Opaque key for stale guards and target lookup. */
  targetKey: FsTargetKey
  /** Path for model/UI-facing output. ... */
  displayPath: string
}

export type FsWriteIntent =
  | { kind: 'createIfAbsent' }
  | { kind: 'replaceIfVersion'; version: FsVersion }
```

`resolve(path)` 把模型给的任意路径解析成稳定的 `FsTarget`（本地实现用 realpath，远程实现可以用文件 id），之后所有操作只收 target。消费方**在类型上就不可能**解析 `targetKey` 或伪造 `FsVersion`——brand 函数注释直说「For backend use only — a consumer never manufactures a key」。同一个文件的所有别名（相对路径、绝对路径、symlink）解析到同一个 key，guard 自然对齐。

`FsWriteIntent` 的设计有个细节值得停一秒：union 只有两个 guarded 分支，「无条件覆盖」不是第三个分支，而是**省略 `expected` 参数**来表达。合同注释：「Omitting the intent from `writeText` means unconditional create-or-overwrite, not a third union arm」。这让「裸 provider 无策略」成为参数缺省时的自然语义，为下文的可卸载策略留好了位。

Definition 还持有三个 `fs/*` 事件的声明（这正是策略插件的接入点）：

```ts
// packages/fs/fs/src/index.ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    /** Single-slot decision for the next writeText ... @mode waterfall */
    'fs/write-intent'(target: FsTarget, actor: object | undefined,
      next: () => FsWriteIntent | undefined | Promise<FsWriteIntent | undefined>): Promise<FsWriteIntent | undefined>
    /** Single-slot decision for the next editText ... @mode waterfall */
    'fs/edit-intent'(target: FsTarget, actor: object | undefined,
      next: () => { version: FsVersion } | undefined | Promise<{ version: FsVersion } | undefined>): Promise<{ version: FsVersion } | undefined>
    /** Record an authoritative positive or negative observation. ... @mode emit */
    'fs/observed'(target: FsTarget, observation: FsObservation, actor: object | undefined): void
  }
}
```

事件只携带 `dsh-fs` 自己的词汇加一个不透明 `object` actor——没有 session、没有 tool 概念，所以发事件的工具包和听事件的策略包互相不依赖，词汇表由 Definition 一家持有。

另外两个容易错过的合同决策：`editText` 是 provider 级原语而不是「读 + 写」的外部组合——版本检查、字面量匹配、原子替换必须共享一个临界区（文件头注释：「`editText` remains here so version check, literal match, and rewrite share one critical section」）；以及 read/write/edit **没有 `timeoutMs` 参数**——`docs/subsystems/filesystem.md` 解释：本地 syscall 最多 best-effort abort，接缝不承诺它无法强制执行的 deadline（对比进程后备的 bash/glob/grep，deadline 能真的杀掉工作）。

### Provider 之一：fs-local 的每目标锁

本地实现里最值得看的是并发控制：

```ts
// packages/fs/fs-local/src/index.ts
/** Per-targetKey tail promise: serializes mutating ops so the read→guard→write
 * window can't interleave, making concurrent writes/edits deterministically
 * ordered (one wins, the rest see the new version and reject as stale). */
private locks = new Map<string, Promise<unknown>>()

private async withLock<T>(targetKey: string, op: () => Promise<T>): Promise<T> {
  const prior = this.locks.get(targetKey) ?? Promise.resolve()
  const run = prior.then(op, op)
  const tail = run.then(() => undefined, () => undefined)
  this.locks.set(targetKey, tail)
  try {
    return await run
  } finally {
    if (this.locks.get(targetKey) === tail) this.locks.delete(targetKey)
  }
}
```

每个 `targetKey` 一条 promise 链就是一把 FIFO 锁：`writeText`/`editText` 的「probe → guard 检查 → 原子写」整段跑在锁内，并发的两个写有确定性顺序——一个赢，另一个在自己的锁窗口里看到新版本、按 `FS_STALE_VERSION` 拒绝。没有互斥量、没有队列类，十几行 promise 组合就把竞态窗口关死了。`editText` 里 guard 检查刻意放在字面量匹配**之前**：过期的编辑报告 `FS_STALE_VERSION` 而不是对着新内容报「找不到 oldString」，模型收到的纠错信号是准的。

### Provider 之二：fs-sandbox = 继承 + 一道围栏

沙箱 provider 不重写任何存储机制：

```ts
// packages/fs/fs-sandbox/src/index.ts
export class SandboxedFileSystem extends LocalFileSystem {
  static inject = ['sandboxPolicy']
  // ...
  private async checkedTarget(target: FsTarget, sandboxPolicy?: SandboxExecutionPolicy): Promise<FsTarget> {
    const policy = sandboxPolicy ?? this.ctx.sandboxPolicy.resolve()
    const { mode } = policy
    if (mode === 'danger-full-access') return target
    if (mode === 'read-only') {
      throw new FsError(`cannot write "${target.displayPath}": file access denied under read-only mode`, 'FS_SANDBOX_DENIED')
    }
    // workspace-write: containment on the FRESH canonical path (catches a
    // symlink ancestor swapped since the tool resolved this target), and the
    // mutation delegates with THIS fresh target — never the stale one.
    const fresh = await this.resolve(target.displayPath)
    let contained = false
    for (const root of writableRoots(policy)) {
      if (await isPathUnder(fresh.targetKey, root)) { contained = true; break }
    }
    if (!contained) {
      throw new FsError(`cannot write "${target.displayPath}": file access denied under workspace-write mode`, 'FS_SANDBOX_DENIED')
    }
    return fresh
  }
}
```

三点：**其一**，`checkedTarget` 返回的是重新 canonicalize 的 fresh target，后续 mutation 用的就是被检查的那个身份——「no check-here-write-there TOCTOU」；祖先 symlink 在工具 resolve 之后被换掉也会被这次 re-resolve 抓到。**其二**，可写根集合来自 `@deepseek-ai/dsh-sandbox` 导出的同一个 `writableRoots()` 函数——bash 沙箱（Seatbelt profile）授予的根和 fs 围栏检查的根**由同一个函数推导**，两个执行家族不可能圈到不同的边界（`docs/capability-seams.md` 对 `ctx.sandboxPolicy` 的注释：「Both enforcing families read it so bash and fs cannot confine to different roots」）。**其三**，模块注释诚实标注威胁模型：这是可信代码里对模型可控路径的策略围栏，不是内核边界；对不可信**代码**的内核级隔离归 `ctx.shell` 的 [bash-sandbox](25-sandbox.md)。拒绝抛结构化的 `FS_SANDBOX_DENIED`——进程内围栏确切知道自己拒了什么，不像 bash 要从 stderr 里猜内核方言。

替换方式也写在类注释里：「loading it INSTEAD OF `dsh-fs-local`, together with a `ctx.sandboxPolicy`, is the whole swap — the model-facing tools are untouched」。上一章的三角模型在此落地。

### 策略：fs-observation-policy 在做什么？

回答标题问题：它维护「谁读过哪个文件的哪个版本」，并据此决定每次写/编辑该带什么 guard。全部状态就是一个双层弱引用表：

```ts
// packages/fs/fs-observation-policy/src/index.ts
class ObservedStateGate {
  private observed = new WeakMap<object, Map<string, FsObservation>>()

  writeIntent(target: FsTarget, actor: object | undefined): FsWriteIntent {
    const owner = this.owner(actor)
    const prior = owner ? this.get(owner, target.targetKey) : undefined
    return prior?.kind === 'present'
      ? { kind: 'replaceIfVersion', version: prior.version }
      : { kind: 'createIfAbsent' }
  }

  editIntent(target: FsTarget, actor: object | undefined): { version: FsVersion } {
    const owner = this.owner(actor)
    const prior = owner ? this.get(owner, target.targetKey) : undefined
    if (!owner || prior === undefined) {
      throw new FsError(`edit requires reading "${target.displayPath}" first`, 'FS_NOT_OBSERVED')
    }
    if (prior.kind === 'absent') {
      throw new FsError(`cannot edit "${target.displayPath}": not found`, 'FS_NOT_FOUND')
    }
    return { version: prior.version }
  }
}

export function apply(ctx: Context): void {
  const gate = new ObservedStateGate()
  // ...
  ctx.on('fs/write-intent', (target, actor) => Promise.resolve().then(() => gate.writeIntent(target, actor)))
  ctx.on('fs/edit-intent', (target, actor) => Promise.resolve().then(() => gate.editIntent(target, actor)))
  ctx.on('fs/observed', (target, observation, actor) => { gate.observe(target, observation, actor) })
}
```

语义表很干净：未见过 → write 走 `createIfAbsent`（防盲覆盖）、edit 直接拒 `FS_NOT_OBSERVED`；确认缺席 → write 仍可 create、edit 拒 `FS_NOT_FOUND`；见过某版本 → 两者都带版本 guard，由 provider 在锁内做最终 CAS。注意**策略只做决定，不做 I/O**——真正的原子检查在 provider 的临界区里，策略只是把「该用什么 guard」注入进去。

owner 的推导也体现了解耦纪律：事件里的 actor 是不透明 `object`，策略用一个结构化弱类型 `FsObservationActor` 把它窄化成 `actor.agent?.session`，拿到的 session 只当 WeakMap 键用、从不读字段——不 import `dsh-tools`、`dsh-agent`、`dsh-session` 任何一个包。WeakMap 让被回收的 session 自动释放观察状态；`apply` 里的 effect 在插件卸载时清空全部状态（HMR 安全）。

三个事件的调度模式选择同样精确：两个 intent 是**单槽 waterfall**——工具 dispatch 时给的默认 thunk 返回 `undefined`（裸 provider 行为），第一个返回值的监听器独占决定权、不调 `next()`；`fs/observed` 是同步 `emit`，因为它在 mutation 已经提交之后触发，监听器必须是同步的纯记录器（合同注释警告：抛错会污染一个已成功的工具结果）。

### Consumer：tool-fs 与它的兄弟们

`tool-fs` 的 `write` 工具把上面所有部件串起来，核心执行路径不到 20 行：

```ts
// packages/fs/tool-fs/src/write.ts
async execute(args: WriteToolArgs, exec) {
  const input = parseWriteArgs(args)
  const sandboxPolicy = await sandbox.resolvePolicy('write', args, exec)
  const target = await ctx.fs.resolve(input.filePath, sessionResolveOptions(exec, input.filePath, sandboxPolicy?.workspaceRoot))
  // Single-slot decision: the policy plugin produces createIfAbsent/
  // replaceIfVersion; the bare default is undefined (unconditional). No stat.
  const intent = await ctx.waterfall('fs/write-intent', target, exec, () => undefined)
  let outcome: FsWriteOutcome
  try {
    outcome = await ctx.fs.writeText(target, input.content, intent, exec.signal, sandboxPolicy)
  } catch (error: unknown) {
    throw remediateFsError(sandbox.mapError(error, sandboxPolicy))
  }
  // Record the present observation (a no-op when no policy plugin listens).
  ctx.emit('fs/observed', target, { kind: 'present', version: outcome.version }, exec)
  return { path: target.displayPath, operation: outcome.operation, before: outcome.before, after: outcome.after }
}
```

工具做的只有：解析参数 → 解析沙箱策略 → resolve target → dispatch 决策 waterfall → 调 provider → emit 观察记录。它不调用策略插件的任何方法（策略根本没有方法可调），卸载 `fs-observation-policy` 后这段代码原样运行，只是 `intent` 恒为 `undefined`、回到无条件写——`docs/subsystems/filesystem.md` 把这一点列为合同：「Removing it does not break the tool」。写成功后自己 emit 一条 present 观察，所以「写过=读过最新版」，连续两次 write 不会被策略拦。

这条接缝还有另外两个消费方，各自展示了不同的接入姿势：

- **`tool-str-replace-editor`**（`packages/fs/tool-str-replace-editor/src/index.ts`）：Anthropic 风格的 `str_replace_editor` 工具方言（view/create/str_replace/insert），同样跑在 `ctx.fs` + `fs/*` 事件之上——同一条接缝可以同时供养两套模型面向的工具 schema，策略状态还是共享的（在 `read` 里看过的文件，`str_replace` 就能编辑）。
- **`tool-fs-search`**（`packages/fs/tool-fs-search/src/index.ts`）：`glob`/`grep` 工具，模块注释专门声明它**故意不 inject `fs`**——「Local workspace discovery is a process-backed `rg` workflow, so these tools execute through `ctx.subprocess.spawn()` with fixed ripgrep argv templates」。搜索是进程后备的工作负载（打包的 ripgrep 二进制），归执行世界接缝管；把它硬塞进 `ctx.fs` 会逼着每个文件系统 provider 实现内容检索。接缝的边界感也体现在「不放什么进来」。

## 设计亮点

> 💎 **设计亮点：策略是事件门，不是服务。** 普通写法会做一个 `ctx.fsPolicy` 服务让工具调用——于是工具永久依赖策略概念，「无策略」还得提供 null 实现。这里策略插件零服务、零 inject，靠占据 waterfall 的单槽存在；卸载它，工具代码一行不改地退回裸 provider 语义。「read-before-write 是部署选择而非工具逻辑」被包结构直接表达了。

> 💎 **设计亮点：branded 不透明类型把合同违规变成编译错误。** `FsTargetKey`/`FsVersion` 是 branded string，消费方无法伪造、无法解析、无法拿本地路径规则去处理远程 key；需要跨能力坐标时必须走 `processPath()`/`fileUrl()`/`contains()` 这三个显式出口。「Consumers MUST NOT parse it」不靠 code review 维持，靠类型系统维持。

> 💎 **设计亮点：guard 分层——策略决定意图，provider 原子执行。** 观察状态在策略层（每 session 一份、弱引用、可整体丢弃），CAS 在 provider 锁内（`withLock` 串行化 read→guard→write）。任何一层单独做都有洞：只有策略层则检查与写入之间有竞态；只有 provider 层则「谁读过什么」无处安放。两层通过 `FsWriteIntent` 这个最小词汇握手。

> 💎 **设计亮点：fs-sandbox 用继承交付「差量 provider」。** 372 行注释加代码里，真正的新逻辑只有 `checkedTarget` 一个函数——存储机制 100% 复用 `LocalFileSystem`，围栏返回 fresh target 消灭 check/use 分离的 TOCTOU，`writableRoots()` 与 bash 沙箱共享同一边界推导。「加一种能力变体 = 继承 + 覆盖两个方法」是接缝三角给 provider 作者的直接红利。

## 小结与延伸

文件系统接缝把一个「先读后写」的产品规则拆成了四个正交部件：Definition 定义不透明身份、版本令牌和三个事件；fs-local 用每目标 promise 锁提供原子 guarded mutation；fs-observation-policy 作为纯事件插件把观察状态折算成 guard 意图；tool-fs 家族只负责 schema、渲染和事件编排。每个部件都能独立替换或卸载，而整体组合起来恰好是 Claude Code 式的文件编辑体验。下一章沿着 `processPath()` 这条线索走进进程一侧，看 fs 与 subprocess 如何构成「同一个执行世界」。

**阅读清单**

- `docs/subsystems/filesystem.md` — 本接缝的完整合同文档（含生成的 Cordis API 目录）
- `packages/fs/fs/src/types.ts` — 全部词汇表与 `FsErrorCode` 错误分类
- `packages/fs/fs-local/src/fsio.ts` — 原子写、UTF-8 流式解码、二进制拒绝的底层实现
- `packages/fs/tool-fs/src/read.ts` 与 `read-render.ts` — 读窗口、行号渲染与 `fs/observed` 的授权语义
- `.agents/notes/implemented/architecture/2026-06-17-filesystem-capability-seam.md` — 这条接缝的决策记录
