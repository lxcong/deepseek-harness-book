# 可逆副作用：Effect 与热重载

前两章反复出现一句话："注册是一个 effect"。这一章正面回答它是什么：`ctx.effect()` 是 Cordis 把一切副作用变成可逆操作的原语——定时器、监听器、服务、HTTP 服务器、prompt section，统统在这个框架下获得"随插件卸载而自动撤销"的能力。我们读 `vendor/cordis/src/fiber.ts` 里 effect 的实现与 teardown 顺序，看 `vendor/hmr/` 如何把"可逆"兑换成"可热替换"，最后从 `packages/` 里找几个真实用例，看 harness 每天怎么使用它。

## 问题背景

插件系统的卸载问题，朴素做法是给每个插件加一个 `onUnload` 钩子，作者在里面手写清理逻辑。实践里它必然腐烂：申请资源的代码和释放资源的代码隔着半个文件，新增一个 `setInterval` 时忘了去 `onUnload` 里补 `clearInterval`；清理顺序全靠作者心算——先关服务器还是先注销路由？异步清理完成前插件算不算卸载完？更糟的是这些错误平时不发作：进程反正要退出，泄漏无人察觉。**直到你想做热重载**——HMR 要求卸载一个插件后世界回到它加载前的状态，任何一处泄漏都会让第二次加载撞上第一次的残骸：端口占用、监听器重复、旧闭包攥着旧服务。

Cordis 的解法是改变申请资源的语法：资源在 `ctx.effect()` 的回调里申请，disposer 在同一个闭包里返回。申请与释放在源码上相邻、在数据结构上配对，框架负责在正确的时机以正确的顺序执行释放。

## 源码剖析

### effect 的核心合同

`docs/cordis-tutorial/02-lifecycle-and-effects.md` 的最小例子先建立直觉：

```ts
// docs/cordis-tutorial/02-lifecycle-and-effects.md
ctx.effect(() => {
  const timer = setInterval(() => console.log('tick'), 200)
  return () => clearInterval(timer)
})
```

实现在 `Fiber.effect()`。剥掉竞态处理的外壳，主干是"收集 disposer，反序执行"：

```ts
// vendor/cordis/src/fiber.ts — effect()（删节）
effect(execute: () => Effect, label = 'anonymous'): any {
  this.assertActive()
  if (this.state === FiberState.UNLOADING) {
    throw new CordisError('INACTIVE_EFFECT')
  }

  const disposables: Disposable[] = []
  let disposing = false
  const dispose = () => {
    if (disposing) return disposalTask
    disposing = true
    let task!: void | Promise<void>
    for (const disposable of disposables.splice(0).reverse()) {
      if (task) {
        task = task.then(() => runDisposable(disposable))
      } else {
        const result = runDisposable(disposable)
        if (isObject(result) && 'then' in result) task = result as any
      }
    }
    return disposalTask = task
  }
  // ...（wrapper 构造、setup 竞态、异步 effect 处理，见后文）
  removeWrapper = this._disposables.push(wrapper)
  task = this._execute(runner)
  // ...
  return wrapper
}
```

几个合同条款直接写在代码里：effect 体在**已在卸载中的 fiber 上创建会抛 `INACTIVE_EFFECT`**——不存在"注册上去但永远不会被清理"的窗口；`disposables.splice(0).reverse()` 保证**同一个 effect 内的多个 disposer 反序执行**；`disposing` 标志让公开的 disposer **幂等**，调两次是 no-op；返回的 wrapper 在 `execute` 抛错时会先回滚已收集的部分再把原错误抛出——**半成品不会留在世界上**。

`execute` 的返回值接受四种形态（`Effect` 类型）：单个 disposer、disposer 的 Promise、以及同步/异步 generator。generator 形态是最常用的进阶写法——每 `yield` 一个 disposer 就登记一个：

```ts
// vendor/cordis/src/fiber.ts — _execute()（删节）
} else if (Symbol.iterator in effect) {
  const iter = effect[Symbol.iterator]()
  while (true) {
    const result = iter.next()
    safeCollect(result.value)
    if (result.done) return
  }
}
```

价值在失败路径：generator 在第三步抛错时，前两步 yield 出的 disposer 已被收集，回滚会执行它们。相比"先做完所有事再一次性返回大 disposer"，generator 把**部分失败的清理**也纳入了框架管辖。

### fiber 卸载：teardown 的两层顺序

单个 effect 内部反序；那 fiber 卸载时，多个 effect 之间呢？

```ts
// vendor/cordis/src/fiber.ts — _unload()（删节）
private async _unload() {
  await Promise.all(this._disposables.clear().map(async (dispose) => {
    try {
      await composeError(async (info) => {
        await Promise.resolve()
        info.error = new Error()
        await runDisposable(dispose)
      }, this._runner.getOuterStack)
    } catch (reason) {
      this.ctx.logger.error(reason)
    }
  }))
  this.store = undefined
  // ...（卸载完成后按 epoch 决定是否立即重载）
}
```

答案是**并发**：fiber 级卸载把所有顶层 disposer 一起启动、`Promise.all` 等齐。教程明文提醒了这个坑："disposers start in reverse registration order, but multiple **async** disposers run concurrently"——如果两步清理必须有先后（先 flush 再关连接），就把它们放进**同一个** effect，用一个 disposer 内部 await 串起来，或靠单 effect 内的反序保证。同时注意每个 disposer 的错误被单独捕获、只记日志：一个 disposer 抛错**不会**阻止其他 disposer 执行，卸载永远走到底。

还有一个不起眼的 WeakMap 值得点名：

```ts
// vendor/cordis/src/fiber.ts
const effectInertia = new WeakMap<Disposable, () => void | Promise<void>>()

function runDisposable(dispose: Disposable) {
  const result = dispose()
  return effectInertia.get(dispose)?.() ?? result
}
```

公开语义上 disposer 是一次性的（第二次调用返回 `undefined`），但 fiber 卸载可能与某个调用方主动 dispose 同一个 effect 并发发生。`effectInertia` 让第二个到场者拿到**第一个到场者正在进行的清理 Promise** 并等它结束，而不是误以为"没事可做"提前宣布卸载完成。可逆性在并发下依然守恒。

### 万物皆 effect

有了这个原语，Cordis 把自己的所有注册 API 都建在上面。[第 5 章](05-typed-events.md)的事件监听：

```ts
// vendor/cordis/src/events.ts — register()
return this.ctx.fiber.effect(() => {
  hooks[method]({ ctx: this.ctx, callback, ...options })
  return () => this.unregister(hooks, callback)
}, label)
```

[第 4 章](04-context-and-service.md)的 `ctx.provide()`、子插件挂载 `ctx.plugin()`（其 dispose 就是父 fiber 上的一个 effect，`fiber.ts` 构造函数里 `this.dispose = parent.fiber.effect(...)`），全部同构。每个 effect 还带 label（`ctx.on("...")`、`ctx.provide("...")`），`fiber.getEffects()` 能吐出一棵带标签的 `EffectMeta` 树——泄漏排查时你看到的不是匿名闭包，而是"哪个插件的哪类注册还活着"。

### HMR：可逆性的兑现时刻

`vendor/hmr/` 是这套设计的验收测试。它监听文件变化，对每个受影响的插件执行"卸载旧的、导入新的、原地重挂"：

```ts
// vendor/hmr/src/index.ts — partialReload()（删节）
const reload = (plugin: any, runtime: Plugin.Runtime) => {
  if (!runtime) return
  for (const oldFiber of runtime.fibers) {
    const fiber = oldFiber.parent.registry.plugin(plugin, oldFiber._config, this.getOuterStack)
    fiber.entry = oldFiber.entry
    if (fiber.entry) fiber.entry.fiber = fiber
  }
}

try {
  for (const [plugin, { filename, runtime }] of reloads) {
    // ...
    this.ctx.registry.delete(plugin)   // 旧 fiber 全部 dispose，effect 反向执行
    reload(attempts[filename], runtime) // 用旧 fiber 的 _config 挂载新导入的模块
  }
} catch {
  rollback() // 恢复模块缓存，重挂旧插件
}
```

注意 HMR 没有为"清理"写一行插件相关的代码。`registry.delete(plugin)` 触发每个旧 fiber 的 dispose，所有 effect 反向解除——监听器、服务、定时器、prompt section 全部消失，依赖这些服务的下游插件按第 4 章的 epoch 机制自动进入 PENDING；新插件挂载后服务重新出现，下游又自动复活。**HMR 的正确性完全外包给了 effect 系统**：只要每个插件的副作用都是 effect，热替换就自动正确；反之，任何一处绕过 effect 的副作用（直接 `setInterval` 不包 `ctx.effect`）都会在第一次热重载时现形。再往上一层，`packages/boot/app-boot` 的 `watchUserPatches` 用同一个 HMR 服务把 `cordis.patch.yml` 也变成热文件（[第 7 章](07-profiles-and-bundles.md)）。

失败路径同样认真：重载抛错时 `rollback()` 恢复 ESM/CJS 模块缓存快照并重挂旧插件——热重载失败的结果是"回到旧版本"，不是"半死状态"。

### harness 里的真实用例

三个例子覆盖三种典型形态。最简单的一行式——persona 插件注册 prompt section：

```ts
// packages/preset/persona/src/index.ts
ctx.effect(() => ctx.systemPrompt.section({
  name: PERSONA_SECTION,
  order: PERSONA_ORDER,
  text: config.text,
  ...(config.complete ? { complete: true } : {}),
}), 'persona.section()')
```

`systemPrompt.section()` 本身返回 disposer，包一层 `ctx.effect` 再给个 label，persona 随预设卸载而从系统提示词里消失。

纯清理型——webserver 关停时要处理 Node 不纳入 `closeAllConnections()` 的升级套接字：

```ts
// packages/host/webserver/src/index.ts
this.ctx.effect(() => async () => {
  const serverClosed = new Promise<void>((resolve) => {
    this.server.close(() => { resolve() })
  })
  this.server.closeAllConnections()
  const upgradedClosed = [...this.upgradedSockets].map(socket => new Promise<void>((resolve) => {
    socket.once('close', () => { resolve() })
    socket.destroy()
  }))
  await Promise.all([serverClosed, ...upgradedClosed])
}, 'webServer.listen')
```

`() => async () => {...}`——effect 体什么都不做，只登记一个异步 disposer。多步关停（关服务器、断普通连接、销毁升级套接字）挤在一个 disposer 里统一 await，正是上文"顺序敏感的清理放同一个 effect"纪律的执行。

generator 型——工具注册表的 `presentAs()`，一次注册横跨两个子系统：

```ts
// packages/core/tools/src/index.ts — presentAs()（删节）
const dispose = ctx.effect(function* (this: ToolRuntime) {
  yield this.layers.effect(ctx, (layer) => {
    if (layer.mode !== undefined) {
      throw new Error(`tools.presentAs("${mode}") conflicts with "${layer.mode}" ...`)
    }
    layer.mode = mode
    return () => { layer.mode = undefined }
  }, { label: 'tools.presentAs()' })
  if (mode !== 'native') {
    yield ctx.systemPrompt.section(this.collapseSection())
    yield ctx.systemPrompt.section(this.sdkSection())
  }
}.bind(this), 'tools.presentAs()')
```

三个 disposer（作用域层的 mode 声明、两个 prompt section）由一个 generator 逐个 yield，对外只暴露一个 disposer。调用方撤销 `presentAs` 时三者反序解除；第二个 yield 抛错时第一个已自动回滚。"一个逻辑动作 = 一个 effect"的粒度设计在这里看得最清楚。

## 设计亮点

> 💎 **设计亮点：申请与释放在同一个闭包里配对**
> `onUnload` 钩子式设计的根本缺陷是申请和释放代码在空间上分离，靠人保持同步。`ctx.effect(() => { const r = acquire(); return () => release(r) })` 让 disposer 天然闭包住它要释放的资源——没有成员变量中转、没有"清理时找不到句柄"、忘写释放在 code review 里一眼可见（一个不返回 disposer 的 effect 长得就可疑）。语法结构本身在执行纪律。

> 💎 **设计亮点：HMR 不含任何清理知识，它只是 dispose + plugin**
> 普通热重载实现要为每类资源写失效逻辑（关连接、清缓存、解绑路由）,永远追不上业务增长。这里 HMR 的核心就一句 `registry.delete(plugin)` 加一句 `registry.plugin(newModule, oldFiber._config)`——清理知识分布在每个插件自己的 effect 里，HMR 只触发机制。新插件想被热重载，不需要注册任何 HMR 回调，只需要"副作用都走 effect"这条它本来就该守的纪律。能力是纪律的免费红利。

> 💎 **设计亮点：generator effect 让"部分完成"也可逆**
> 多步注册里第 N 步失败，前 N-1 步怎么办？朴素代码要么泄漏、要么手写 try/finally 金字塔。generator 形态下每个 yield 即刻登记，`_execute` 与 wrapper 的回滚路径保证异常时已 yield 的部分反序解除。`tools.presentAs()` 三步注册的原子性没花作者一行清理代码。

> 💎 **设计亮点：teardown 的错误被隔离，卸载永远走到底**
> `_unload` 对每个 disposer 单独 try/catch、只记日志。反面写法（一错即停）会让一个坏 disposer 阻塞整棵子树的卸载，HMR 从此卡死。这里的取舍很清醒：卸载路径上，"继续拆"永远比"停下来"损害小——与 [第 5 章](05-typed-events.md) parallel 模式的 `allSettled` 是同一个哲学。

## 小结与延伸

`ctx.effect()` 把"副作用"从随手写的过程代码升格为带生命周期、带标签、带失败回滚的一等对象；fiber 卸载给出两层顺序保证（effect 内反序、effect 间并发）；HMR 则证明了这套可逆性的含金量——热替换一个插件不需要该插件的任何配合。这个原语还会继续出场：[第 11 章](../part3/11-scope.md)的 per-agent 作用域注册建立在 scoped context 的 effect 上，[第 34 章](../part9/34-boot.md)里 boot 失败时对半成品插件树的整体 dispose 也是同一机制。

**阅读清单**

- `vendor/cordis/src/fiber.ts` — `effect()`/`_execute()`/`_unload()`，本章所有竞态注释的原文
- `docs/cordis-tutorial/02-lifecycle-and-effects.md` — 可跑的最小 effect 例子与 fiber 状态机
- `docs/cordis-tutorial/06-composition-and-hmr.md` — HMR 上手与 PENDING 诊断
- `vendor/hmr/src/index.ts` — `partialReload()` 全文，含模块缓存快照与回滚
- `packages/core/tools/src/index.ts` 的 `presentAs()`、`packages/host/webserver/src/index.ts` 的关停 effect — 生产级用例
- `docs/cordis-primer.md` 最后一节 — "Every registration should have a disposer" 的官方表述
