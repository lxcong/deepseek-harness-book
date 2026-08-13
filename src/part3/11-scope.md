# 第 11 章 Scope:per-agent 注册的作用域原语

同一个进程里跑着十个 agent,每个要有自己的工具集、自己的 prompt 段落、自己的事件监听器,互相不可见,agent 销毁时各自的注册要干净退场——这是多 agent harness 的地基问题。`packages/core/scope` 用不到 500 行(`index.ts` 204 行 + `store.ts` 267 行)给出一个不依赖任何其他 harness 包的库原语:一个不透明的身份 key、一个打了标签的注册 context、一个只管路由的事件载体,以及注册表共用的分层存储。本章读完你会明白 `agent.ctx` 为什么「注册进去的东西只属于这个 agent」,以及这如何成为「给一个 session 单独一套能力集」的基础。

## 问题背景

朴素做法有两种,各有各的死法:

1. **注册时带 agentId 参数**:`ctx.tools.register(tool, { agentId })`。每个注册表都要加这个参数、都要自己实现按 id 过滤、都要自己处理 agent 销毁时的批量清理;事件系统还得另想办法。参数会传丢,清理会漏,十个注册表十套过滤代码。
2. **每个 agent 一套全新容器**:为每个 agent 起一个独立的 DI 容器/Context 树。隔离是干净了,但全局注册(所有 agent 共享的工具、host 级监听器)反而无处安放,跨 agent 观察(父 agent 看子 agent 的事件)要打洞。

Harness 的解法基于 Cordis 已有的两个机制(见[第 4 章](../part2/04-context-and-service.md)与[第 6 章](../part2/06-reversible-effects.md)):context 可以 `extend()` 出携带附加属性的视图,effect 挂在 fiber 上、fiber 销毁时自动逆转。scope 包在此之上只补三样东西:**身份**(ScopeKey)、**标签的读写**(createScope/scopeOf)、**按标签路由**(scopeTarget + ScopedLayers)。隔离与清理都不用重新发明。

## 源码剖析

### ScopeKey:身份就是对象本身

```ts
// packages/core/scope/src/index.ts
/** An opaque, identity-compared scope key. */
export type ScopeKey = object

/** Context tag written by {@link createScope}. */
const kScope = Symbol('dsh.scope')
```

`ScopeKey` 就是任意对象,按引用比较。没有注册表、没有 id 分配、没有字符串命名冲突。shipped loop 直接**拿 Agent 对象自己当 key**(`docs/subsystems/scope.md`:"The shipped loop uses the live `Agent` object as its own key, but the primitive never inspects the object")——身份与生命周期天然绑定,agent 对象被回收,key 也就没了。

### createScope:一个 no-op 插件 + 一个 symbol 标签

```ts
// packages/core/scope/src/index.ts
/** Shared no-op plugin used as the backing scope fiber. */
function scope(): void {}

export function createScope(ctx: Context, key: ScopeKey, options?: CreateScopeOptions): Scope {
  if (options?.parent !== undefined) bindScopeParent(key, options.parent)
  const fiber = ctx.plugin(scope)
  const scoped: Context = fiber.ctx.extend({ [kScope]: key })
  let disposing: Promise<void> | undefined
  return {
    ctx: scoped,
    rawDispose: fiber.dispose,
    dispose: () => (disposing ??= quiesceFiber(fiber)),
  }
}

export function scopeOf(ctx: Context): ScopeKey | undefined {
  return (ctx as Context & { [kScope]?: ScopeKey })[kScope]
}
```

实现薄得惊人:装载一个什么都不做的插件拿到一根 fiber,再把 fiber 的 context `extend` 上 `{ [kScope]: key }`。但两半各司其职,合起来恰好是「per-agent 注册」的完整语义:

- **fiber 负责生命周期**。通过 `scoped.ctx` 做的一切注册(effect、监听器、服务贡献)都挂在这根 fiber 上;`dispose()` 走 `quiesceFiber` 把它们全部逆转,还处理了 Cordis 的异步拆除惯性(`while (fiber.inertia !== undefined) await fiber.inertia`)。agent 销毁 = 一次 fiber 销毁,零个「忘了清理」的角落。`dispose` 用 `??=` 记忆化,并发的拆除方等待同一次收敛;`rawDispose` 则保留 Cordis 精确的 disposer 身份,供有序复合 effect 嵌套(第 12 章会看到 agent-loop 靠它编排拆除顺序)。
- **标签负责可见性**。`kScope` 是模块私有 symbol,外界只能通过 `scopeOf()` 读,不能伪造写入;沿 context 原型链继承,所以从 `agent.ctx` 再 `extend`/`inject` 出来的任何子 context 天然带着同一个标签。

`ReactLoopAgent` 的构造函数把两者接上 agent:

```ts
// packages/core/agent-loop/src/agent.ts
this.scope = createScope(loopCtx, this)
this.ctx = this.scope.ctx.extend({ agent: this })
```

`agent.ctx` = scope 标签(注册归属)+ `agent` 属性(让代码能从 context 反查到 agent)。全局 `ctx` 与 `agent.ctx` 的关系由此清晰:**同一棵 Cordis 树,不同的视图**——`agent.ctx` 只是多了一个 symbol 标签的 context,访问的服务、事件系统与全局完全相同,差别只在注册时被打上了谁的名字。

### scopeTarget:事件按标签路由,且只向上流

注册隔离解决了「谁的贡献」,事件路由解决「谁能听见」。`scopeTarget` 造出一个只管路由的载体(carrier):

```ts
// packages/core/scope/src/index.ts
export function scopeTarget<T extends object>(base: T, key: ScopeKey | undefined): Scoped<T> {
  const baseFilter = (base as { [CordisContext.filter]?: (ctx: Context) => boolean })[CordisContext.filter]
  const carrier = {
    [CordisContext.filter](ctx: Context): boolean {
      if (baseFilter !== undefined && !baseFilter.call(base, ctx)) return false
      const tag = scopeOf(ctx)
      if (tag === undefined) return true
      for (let cursor = key; cursor !== undefined; cursor = scopeParents.get(cursor)) {
        if (cursor === tag) return true
      }
      return false
    },
  }
  carrierKeys.set(carrier, key)
  return carrier as unknown as Scoped<T>
}
```

派发 `agent/*` 事件时,dispatcher 把这个 carrier 当 `this` 传入(第 9 章读过的 `agentEvents` 就是干这个的),Cordis 对每个监听器调用 filter:**未打标签的监听器(全局注册)什么都收**;打了标签的监听器,只有当标签等于派发 key **或其某个祖先**时才收。注释里那句话值得抄下来:"events flow up the chain, never down"——父 scope 的监听器能看到所有子孙 agent 的事件(一个 standing composition 观察它旗下每个 agent),子 scope 却看不到兄弟或父级的,隔离方向不对称且正确。

祖先关系由一个模块级 WeakMap 维护,同一条关系同时供两个方向使用——这是 `scopeParents` 上那段注释点破的对称性:注册视图**向下**继承(子 scope 看得见祖先的 layer),事件准入**向上**延伸(祖先的监听器收得到子孙的事件)。写入端有讲究:

```ts
// packages/core/scope/src/index.ts
export function bindScopeParent(key: ScopeKey, parent: ScopeKey): ScopeParentBinding {
  if (scopeParents.has(key)) {
    throw new Error('dsh-scope: scope key is already bound to a parent; re-linking requires the binding returned by the original bind')
  }
  linkScopeParent(key, parent)   // 内含成环检查
  return {
    rebind(next: ScopeKey): void { linkScopeParent(key, next) },
  }
}
```

一个 key 只能绑一次父级;想改绑,唯一的路是最初那次 `bindScopeParent` 返回的 `ScopeParentBinding` 对象——**改链能力是一个 capability**,不是公开 API。谁第一个绑定,谁独占重绑权。

### ScopedLayers:注册表的共享分层存储

工具注册表、prompt 段落注册表都面临同一个问题:全局一层 + 每个 scope 一层,读取时按「全局 → 远祖先 → …… → 本 scope」合并,近者胜出。`store.ts` 把这套逻辑写成一份共享实现:

```ts
// packages/core/scope/src/store.ts
merge<V>(
  scope: ScopeKey | undefined,
  pick: (layer: L) => NamedEntries<V>,
): Map<string, V> {
  const merged = new Map(pick(this.global).entries())
  for (const layer of this.chainLayers(scope)) {
    for (const [name, value] of pick(layer).entries()) merged.set(name, value)
  }
  return merged
}
```

`chainLayers` 返回的顺序是远祖先在前、精确 scope 在后,所以后写入 `merged` 的近层会覆盖同名条目——agent 本地注册的工具可以 shadow 全局同名工具。与之相对,`peek(scope)` 刻意**不走链**(注释:"Deliberately chain-blind"),供「只看这个 scope 自己的贡献」的场景(比如它自己的限制、守卫)使用:继承是否生效,由调用方按语义显式选择,而不是存储层替你决定。

写入端 `effect(ctx, action, options)` 从 context 读 `scopeOf(ctx)` 决定落哪一层,同时把 undo 挂到**同一个 context** 的 fiber 上——「注册到哪个 scope」与「随哪个生命周期清理」由一个参数同时决定,不可能岔开。scoped layer 在完全清空时被回收,长期运行的进程不会积累空壳。

消费端的真实接线可以在两处看到:`packages/core/tools/src/index.ts` 用 `new ScopedLayers(...)` + `chainLayers(exec.agent)` 实现工具的 per-agent 视图([第 18 章](../part5/18-tool-registry.md)),`packages/core/system-prompt/src/index.ts` 同样用它实现 prompt 段落的 scope 合并([第 21 章](../part5/21-system-prompt.md))。两个注册表零重复代码。

### 「给一个 session 单独一套能力集」

把上面四件零件合起来,就是 preset 系统的地基:`ctx.agentPresets.mount(agentCtx, id)` 先保证 preset 的 standing composition 挂载,然后**把 agent 的 scope key parent 到那个 composition 的 key 上**——从此该 composition 注册的工具、prompt 段落、监听器,通过链式继承覆盖这个 agent;子 agent 则用 `composeFrom(agentCtx, parentCtx)` 直接接到父 agent 已在跑的那一代 composition 上(同步、不读文件,所以能在创建窗口里用)。preset 还把服务发布在 Cordis 的 `isolate` realm 后面,使两个 session 各自的服务实例彼此不可见——realm 机制与审批预设的细节留给[第 20 章](../part5/20-approval-and-presets.md)和[第 30 章](../part8/30-subagent.md),这里只需要看清:**scope 的父链提供「谁看得见什么」,isolate realm 提供「服务实例归谁」,两者合成 per-session 能力集**。

## 设计亮点

> 💎 **设计亮点:对象即身份,WeakMap 即注册表**
> `ScopeKey = object` 加三个模块级 WeakMap(`carrierKeys`、`scopeParents`、context 上的 symbol 标签),整个原语没有一处需要「注销」的全局状态——agent 对象不可达时,它的父链关系、carrier 关联随 GC 自然消失。换成字符串 id + Map 的普通写法,每一处都是一个潜在泄漏点,还要处理 id 冲突与伪造。
>
> 同一个 WeakMap(`scopeParents`)服务两个方向——注册视图向下继承、事件准入向上延伸——一条关系,两种消费。

> 💎 **设计亮点:一个 context 参数同时决定可见性与生命周期**
> `ScopedLayers.effect(ctx, ...)` 从 `ctx` 读 scope 定归属层,又把 undo 挂上 `ctx` 的 fiber。调用方拿着 `agent.ctx` 注册工具,「只有这个 agent 看得见」和「agent 销毁时自动注销」是同一个动作的两面,不存在「注册对了却清理错了」的状态空间。这是把 Cordis 的可逆 effect([第 6 章](../part2/06-reversible-effects.md))与作用域标签焊在一起的收益。

> 💎 **设计亮点:重绑父链是返回值,不是 API**
> `bindScopeParent` 只许绑一次,重绑能力藏在返回的 `ScopeParentBinding` 里——capability 风格的权限设计:preset 注册表(recompose 的合法执行者)拿着 binding,其他任何代码拿不到,也就不可能把一个 agent 的能力集偷偷改接到别的 composition 上。成环检查在 bind 与 rebind 共用一个 `linkScopeParent`,不变量只写一遍。

## 小结与延伸

scope 包用「对象当 key、symbol 当标签、no-op fiber 当生命周期锚点、WeakMap 当父链」四个极轻的机制,把 per-agent 注册做成了所有注册表可复用的库原语:`agent.ctx` 与全局 `ctx` 是同一棵树的两个视图,事件沿父链向上可见、注册沿父链向下继承,preset 的 mount/composeFrom 在这条链上给每个 session 拼出独立能力集。它是[第 18 章](../part5/18-tool-registry.md)工具注册表与[第 21 章](../part5/21-system-prompt.md) prompt 组装的公共下层,也是[第 30 章](../part8/30-subagent.md)子 agent 继承父能力的机制来源。

**阅读清单**
- `packages/core/scope/src/index.ts` 与 `src/store.ts` — 全部源码不到 500 行
- `packages/core/scope/README.md` — 可调用 API 与过滤语义的权威文档
- `docs/subsystems/scope.md` — 类型层面的浓缩版
- `.agents/notes/implemented/architecture/2026-07-12-agent-scope-runtime-design.md` — 「一个不透明 key 选择一个 layer」的设计笔记
- `packages/preset/agent-presets/src/index.ts` — `mount`/`composeFrom`/`recompose`:scope 父链的最大消费方
