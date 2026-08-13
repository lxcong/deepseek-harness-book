# 第 2 章 仓库导览：monorepo 的组织方式

这一章带你把 deepseek-harness 仓库从根目录到 `docs/` 完整走一遍。目标不只是记住「什么在哪」，而是学会一套方法：拿到一个几十个包的大型 monorepo，先看什么、按什么顺序看，才能在一小时内建立可靠的方位感。dsh 的仓库布局本身就是精心设计过的教材——分组逻辑、workspace 声明、脚本命名全都在向读者传递信息。本章的每个目录清单都用 `ls` 实际验证过。

## 问题背景：迷失在四十个目录里

朴素的读法是 `git clone` 之后直接钻进 `packages/` 挨个点开 `src/`。在一个只有五个包的仓库里这行得通；在 dsh 这种 `packages/` 下有 45 个分组、每组又含多个包的仓库里，这样读半天连「哪个包是入口」都找不到。另一个常见错误是把 README 当营销材料跳过——在工程文化好的仓库里，README 恰恰是导航系统本身。

正确的顺序是自顶向下的四步：**根目录清单 → workspace 声明 → 根 `package.json` scripts → 分层的 README/AGENTS**。前三步告诉你这个仓库「有哪些世界、怎么构建、作者最在乎什么」，第四步才进入具体代码。下面按这个顺序走。

## 第一步：根目录一眼

根目录的 `AGENTS.md` 直接给出了一张带注释的布局图（这本身就是个信号：这个仓库预期被 agent 和人共同阅读）：

```text
# AGENTS.md（节选）
vendor/      Vendored Cordis source — manifest + sync procedure in vendor/README.md
packages/    @deepseek-ai/dsh-<pkg> workspaces at packages/<group>/<pkg>/
python/      Python SDK and bundled runtime (see python/README.md)
native/      @deepseek-ai/node-addon-landlock-run source of record (see native/README.md)
examples/    Runnable cordis.yml leaves over packages/examples bundles
.agents/     Agent workflows and Agent Notes (`notes/`)
docs/        architecture, generated catalogs, postmortems, cookbook (see docs/AGENTS.md)
scripts/     repo gates and generators
website/     VitePress projection of selected bilingual docs/ sources
```

再配合 `ls` 根目录，几类文件值得注意：七份 `vitest.*.config.ts`（unit/e2e/snapshot/web/web-perf/web-stress，测试被切成多个独立世界，[第 38 章](../part10/38-testing-strategy.md)专讲）；四份 `tsconfig.base*.json` / `tsconfig.host.json` / `tsconfig.client.json`（编译分「面」，见下文 scripts 一节）；成对出现的 `README.md` / `README.zh.md` / `README.i18n.yaml`（双语文档流水线，见 docs 一节）。

## 第二步：packages/ 的分组逻辑

`ls packages/` 得到 45 个条目。分组规则是 `packages/<group>/<pkg>/`，npm 包名统一为 `@deepseek-ai/dsh-<pkg>`（包名不含组名——组是阅读结构，不是命名空间）。`packages/README.md` 用一张表给每个组标注了角色和发布预期（Product / POC / Support），并且规定：**每个组的 README 拥有该组的「包 → ctx key」对照表**。比如 `packages/core/README.md`：

```markdown
| Package | Role | ctx key |
|---|---|---|
| [`session/`](session/README.md) | Event-sourced session log and in-memory store | `ctx.sessions` |
| [`system-prompt/`](system-prompt/README.md) | Prompt and tool-schema assembly registry | `ctx.systemPrompt` |
| [`tools/`](tools/README.md) | Scoped tool registry and execution pipeline | `ctx.tools` |
| [`agent/`](agent/README.md) | Agent interface, registry, and event vocabulary | `ctx.agents` |
| [`agent-loop/`](agent-loop/README.md) | Default concrete agent driver | `ctx.agentLoop` |
```

分组的真正逻辑不是「按技术层」，而是**按能力接缝（seam）分家**：一个组就是一个能力家族，Service Definition、各个 Provider、Consumer 工具住在同一组的不同包里。用 `ls` 验证两个典型组：

```text
$ ls packages/fs
fs  fs-local  fs-observation-policy  fs-sandbox
tool-fs  tool-fs-search  tool-str-replace-editor

$ ls packages/shell
shell  shell-env  bash-local  bash-sandbox  pwsh-local  pwsh-sandbox
tool-bash  tool-bash-persistent  tool-pwsh
```

命名即架构：`fs` 是 Service Definition，`fs-local` / `fs-sandbox` 是 provider，`tool-fs*` 是模型可见的 Consumer。看到一个 `tool-` 前缀你就知道它消费某个 seam，看到 `-local` / `-sandbox` 后缀你就知道它是可替换实现。同样的三段式在 `llm/`（`llm` + `llm-deepseek` / `llm-pi-ai` + `llm-retry` / `token-meter`）、`subprocess/`、`terminal/`、`skill/`、`web/` 等组里反复出现。少数组不是 seam 而是横切设施：`core/`（产品 API 主轴）、`bundle/`（发行层叠）、`boot/`（启动胶水）、`util/`（零依赖工具）、`test-support/`。

组间依赖关系不用脑补——`docs/module-graph.md` 是生成的全量依赖图（下文还会讲到它）。依赖的纪律只有一条，`packages/README.md` 原文加粗：「Extension plugins depend on Service Definitions, never concrete providers.」

## 第三步：apps/ 与 packages/ 的关系

`apps/` 只有两个成员：`cli` 和 `web`。`apps/cli/package.json` 的名字就是发布名 `@deepseek-ai/dsh`，`bin` 字段指向 `dsh` 命令。打开它的 `dependencies` 会发现几乎全是 `workspace:^` 引用——**app 不写业务逻辑，app 是包的装配清单**。`apps/web` 同理，只是浏览器前端的构建壳（`vite.config.ts` + `index.html`），真正的 UI 代码在 `packages/client/` 和 `packages/host/`。这是 monorepo 的一条通用鉴别法：`apps/` 越薄，说明抽象做得越到位；如果 `apps/` 里有几千行 `src/`，那才要警惕。

## 第四步：vendor/、native/、python/ 的角色

这三个目录是 dsh 区别于普通 TypeScript monorepo 的地方。

**`vendor/`** 装的是整个 Cordis 框架的源码拷贝：`cordis`、`loader`、`hmr`、`timer` 等九个包，全部改挂到 `@deepseek-ai` scope 下（`cordis` → `@deepseek-ai/cordis`）。`vendor/README.md` 维护一张 manifest，逐包记录上游仓库、版本、commit SHA 和本地修改日志。为什么不直接 npm 依赖？README 一句话说透：「so that the harness fully owns its framework layer (auditable, patchable, pinned)」。改名则是因为 harness 发布时会连框架层一起发布，用上游原名会在 registry 上抢注别人的名字。这个决策的完整展开在[第 42 章](../part10/42-vendoring.md)。

**`native/`** 只有一个成员 `landlock-run`：Linux Landlock「先自我约束再 exec」的原生启动器，是沙箱能力（[第 25 章](../part6/25-sandbox.md)）的底层依赖。它有独立的三包 npm 发布族和自己的 release 流程，但源码和 harness 消费方同仓共进——「a launcher contract change and its consumer update can land and be tested together」。

**`python/`** 是 Python SDK 两件套：`sdk`（高层 turns API + 底层 JSON-RPC 客户端）和 `sdk-runtime`（打包好的运行时二进制）。Python 侧不重新实现任何 harness 逻辑，只是把 dsh 作为子进程拉起来，通过 stdio 上的 newline-delimited JSON-RPC 驱动——又一次印证第 1 章的结论：所有集成面都是同一个内核的不同投影。

## 第五步：pnpm-workspace.yaml 里的设计

workspace 声明文件通常无聊，dsh 的这份值得逐段读。三处最有信息量：

```yaml
# pnpm-workspace.yaml（节选，注释为原文）
packages:
  - vendor/*
  - packages/*/*
  # Members for DEPENDENCY RESOLUTION only — NOT build targets: ...
  - examples
  # Deploy root of the single-exe build: a pure dependency manifest whose
  # closure is what the exe bundles and what the Python runtime distributes.
  - python/sdk-runtime

# Vendored framework packages keep their upstream semver ranges, while local
# builds must resolve those matching names to this workspace's pinned sources.
linkWorkspacePackages: true

allowBuilds:
  esbuild: true
  # Pulled in by @earendil-works/pi-ai (optional LLM API backend). ...
  # ... those are no-ops we don't need, so we deny them — install still succeeds.
  '@google/genai': false
  protobufjs: false
```

第一，`linkWorkspacePackages: true` 是 vendoring 方案的另一半：vendored 包保留上游 semver 版本号，这个开关强制所有对这些名字的解析都落到仓库内的 pinned 源码，而不是 registry。第二，`examples` 作为 workspace 成员存在的唯一目的是依赖解析（让示例的 `cordis.yml` 能通过真实的包 `exports` 解析插件），注释明确说它不是构建目标。第三，`allowBuilds` 是 pnpm 10+ 的安装脚本白名单，dsh 的策略是**默认拒绝、逐包审查、每个例外写明理由**——连「拒绝但安装仍成功」这种细节都写在注释里。workspace 文件被当成了设计文档来写。

## 第六步：根 package.json 的 scripts 词法

scripts 有一百多条，但命名有严格的词法，认得词根就能猜出全貌：

```jsonc
// package.json scripts（节选）
"build:lib:host": "tsc -b tsconfig.host.json && tsdown --env.DSH_BUILD_FACE host",
"build:lib:client": "tsc -b tsconfig.client.json && tsdown --env.DSH_BUILD_FACE client",
"gen-module-graph": "tsx scripts/gen-module-graph.ts",
"verify-module-graph": "tsx scripts/gen-module-graph.ts --check",
"gen-tool-catalog": "tsx scripts/gen-tool-catalog.ts",
"verify-tool-catalog": "tsx scripts/gen-tool-catalog.ts --check",
"doc-sync": "tsx scripts/run-gates.ts doc-sync",
"check:ci": "tsx scripts/run-gates.ts ci-primary",
"dsh": "node --import tsx/esm apps/cli/src/bin.ts",
```

四个观察。其一，构建分两个「面」（face）：host（Node 侧）和 client（浏览器侧），各有独立的 tsconfig 与 tsdown 入口，源码平面与产物平面绝不混用（`AGENTS.md`：「Source plane vs artifact plane, never mixed」）。其二，`gen-*` / `verify-*` 永远成对，且 verify 就是同一个脚本加 `--check`——生成器与校验器共享实现，不可能漂移。其三，几十个 `verify-*` 叶子门禁被 `scripts/run-gates.ts` 聚合成 `doc-sync`、`check:ci` 等套餐，CI 与本地跑同一份编排。其四，`pnpm dsh` 直接用 tsx 从源码启动 CLI，源码即可运行。另外留意 `test:coverage`：根 `AGENTS.md` 注明它是 CI 覆盖率门禁，标准是 `packages/*/*/src` **每文件 100%**。

## 第七步：docs/ 的文档体系

`docs/` 不是一堆散文，而是一个有类型系统的文档工程。`docs/AGENTS.md` 定义了「tier 分层」：每个事实只有一个家（one home per fact），root `AGENTS.md` 放常备规则、`architecture.md` 放行为总图、`subsystems/` 每个子系统一页参考（含生成的 Cordis API 区块）、Agent Notes 放决策理由，越界内容必须改成链接。主要成员：

- **`docs/subsystems/`** — 40+ 页,一个子系统一页,是查类型定义和事件语义的第一站；
- **`docs/cookbook/`** — 任务导向的操作指南：adding-a-package、adding-a-tool、adding-an-llm-adapter、adding-a-conversation-node、adding-a-vendored-package；
- **`docs/cordis-primer.md` + `docs/cordis-tutorial/`** — Cordis 速成页与 7 步上手教程（`01-first-plugin` 到 `07-into-the-harness`），本书第二部分的官方预习材料；
- **`docs/postmortem/`** — 四份事故复盘，编号存档（[第 40 章](../part10/40-docs-as-engineering.md)详读）；
- **生成式文档** — `module-graph.md`、`tool-catalog.md`、`config-catalog.md`、`event-producer-consumer.md` 等，头部一律带着自己的出处：

```html
<!-- Generated by scripts/gen-module-graph.ts — do not edit by hand.
     Run `pnpm run gen-module-graph` to regenerate. -->
```

- **i18n 三件套** — 每篇文档都是 `.md` / `.zh.md` / `.i18n.yaml` 三个文件，YAML 记录中英段落配对，`verify-translation-pairing` 门禁保证双语不脱节（[第 40 章](../part10/40-docs-as-engineering.md)）。

生成式文档配上 `verify-*` freshness 门禁，意味着这些「文档」实际上是 CI 保鲜的数据库视图——读它们比读代码更快，且不会过期。

> 💎 **设计亮点：目录命名承载架构语义。** 普通 monorepo 里包名只是标签，读者要打开 `src/` 才知道一个包是接口还是实现；dsh 用 `<seam>` / `<seam>-<impl>` / `tool-<seam>` 的命名词法把 seam 三角直接写进目录树，`ls packages/fs` 的输出本身就是一张架构图。信息密度最高的文档是文件系统本身。

> 💎 **设计亮点：gen 与 verify 是同一个脚本。** 生成式文档最大的风险是「生成后被手改、或源变了忘了重新生成」。dsh 把校验做成 `gen-*.ts --check`——校验器复用生成器的全部逻辑，只比对输出是否一致，再由 CI 的 `doc-sync` 门禁强制执行。相比另写一个独立 checker，这个做法从构造上排除了「生成器与校验器规则不一致」这类二阶漂移。

> 💎 **设计亮点：把 workspace 文件写成审计记录。** `pnpm-workspace.yaml` 的 `allowBuilds` 段默认拒绝一切安装脚本，每个 `true` / `false` 都附带一句为什么：node-pty 需要跨平台 PTY 构建、`@google/genai` 的脚本是 no-op 所以拒绝。供应链决策不藏在某次 PR 讨论里，而是永久驻留在决策生效的那一行旁边。

## 小结与延伸

读大型 monorepo 的顺序是自顶向下：根目录清单定世界观，workspace 声明定成员关系，根 scripts 定构建与门禁词法，分层 README/AGENTS 定每个事实的家。dsh 在每一层都把「布局」当成表达设计的机会：分组词法写出 seam 三角，vendor 目录写出框架所有权，`gen`/`verify` 成对写出文档保鲜机制。带着这张地图，下一章我们进入 `docs/architecture.md` 描绘的运行时结构：插件树、事件域与 turn flow。

**阅读清单**（相对仓库根）：

- `AGENTS.md` — 布局图 + 全部工程纪律的索引
- `packages/README.md` — 45 个组的角色与发布预期总表
- `packages/core/README.md`、`packages/shell/README.md` — 两份典型的组级 ctx-key 地图
- `pnpm-workspace.yaml`、`package.json` — 本章第五、六步的原文
- `docs/AGENTS.md` — 文档 tier 分层与「one home per fact」
- `docs/module-graph.md` — 生成的包依赖全图，配合本章的分组表食用
