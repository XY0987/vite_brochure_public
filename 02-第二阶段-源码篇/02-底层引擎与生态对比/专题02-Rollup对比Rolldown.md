# 专题 02 · Rollup vs Rolldown 深度对比：同一套 API，两种实现，两种命运

> 本节目标：把Rollup vs Rolldown 对比表「讲透到实现层」——它们为什么一个是基准、一个快 10–30x，为什么 Rolldown 能「兼容 Rollup 插件 API 却又不完全兼容」，以及在 Vite 8.1.0 源码里它具体出现在哪三个地方、`rollupOptions` 是怎么被代理成 `rolldownOptions` 的。
>
> 源码基线：Vite 8.1.0，`rolldown@~1.1.2`。锚点：`packages/vite/src/node/build.ts`、`utils.ts`、`optimizer/index.ts`、`config.ts`。

---

## 一、先看对比（带实现层注解）

| 维度 | Rollup | Rolldown | 实现层注解 |
|---|---|---|---|
| 实现语言 | JavaScript | Rust | 语言决定了性能天花板与分发方式（见下文「三、为什么是 10–30x：Rust 带来的不只是『语言更快』」和「四、语言差异的『隐藏成本』：分发与调试」） |
| 定位 | 生产构建打包器 | 统一 dev + build 的单一打包器 | Rolldown 既要做 build 打包，又要做 dev 预构建 |
| 速度 | 基准 | 构建快约 10–30x | Rust + 原生并行；具体倍数依项目而定 |
| 插件生态 | 原生 | 兼容 Rollup 插件 API（依赖rollup内部实现的可能失效） | 兼容的是「公开钩子契约」，不是「内部实现」（[专题 04 生态连带影响（实现层）](./专题04-生态连带影响.md) 展开） |
| 配置入口 | `build.rollupOptions` | `build.rolldownOptions`（含兼容层自动转换） | 源码里用 `setupRollupOptionCompat` 做 getter/setter 代理 |
| 在 Vite 中 | Vite 2–7 的 build 引擎 | Vite 8 默认唯一引擎 | Vite 8 里 `rollup` 退为 devDependency，`rolldown` 才是运行时依赖 |

下面逐行展开最值得讲的几点。

---

## 二、定位的根本差异：从「只管 build」到「dev/build 通吃」

这是两者最容易被忽略、却最重要的区别。

- **Rollup 的定位始终是「生产打包器」**：它只做一件事——把整张依赖图一次性打包成优化产物。dev 期 Vite 从来不用 Rollup（dev 用 esbuild，见 [专题 01 双引擎的历史与代价](./专题01-双引擎的历史与代价.md)）。
- **Rolldown 的定位是「统一打包器」**：它不仅替代 Rollup 做 build 打包，还要接管 dev 期的**依赖预构建**（原来 esbuild 的活）和**配置文件打包**。

所以 Rolldown 在 Vite 8 里出现的位置比 Rollup 当年多得多。这正是「双引擎 → 单引擎」收敛的实现落点：**同一个打包器，dev 和 build 都用它，行为天然更一致。**

---

## 三、为什么是 10–30x：Rust 带来的不只是「语言更快」

「Rust 比 JS 快」只是表层。Rolldown 的速度优势来自三层叠加：

1. **原生执行**：Rust 编译成机器码，没有 JS 的解释/JIT 预热开销，解析、转换、生成各阶段都更快。
2. **真并行**：Node.js 的 JS 主线程是单线程的，Rollup 很难吃满多核；Rolldown 在 Rust 侧用多线程并行处理模块，CPU 越多核优势越明显。
3. **同源工具链零序列化开销**：Rolldown 内嵌 Oxc（同为 Rust），解析→转译→打包在同一个 Rust 进程内用同一套 AST 流转，省掉了「JS 工具之间反复传字符串/序列化 AST」的成本（[专题 03 Oxc 工具链与边界](./专题03-Oxc工具链与边界.md) 展开）。

> 「10–30x」是一个**区间而非定值**。小项目可能感知不强（启动/IO 占比高），大型 monorepo、依赖众多的项目收益最明显。前面说的快「约 10–30x」并强调「依项目而定」，正是这个原因——别把区间值当成对任意项目的承诺。

---

## 四、语言差异的「隐藏成本」：分发与调试

Rust 不是只有好处。换成原生打包器，代价体现在两处：

| 方面 | JS 版 Rollup | Rust 版 Rolldown |
|---|---|---|
| 安装/分发 | 纯 JS，跨平台一份代码 | 需按平台分发预编译二进制（platform-specific binary） |
| 调试 | 可直接在 JS 里断点、改源码 | 核心逻辑在 Rust 侧，JS 层只能看到入参/出参，深入要切到 Rust 调试 |
| 安装体积 | 较小 | 含原生二进制，体积更大 |

这也是为什么本书的源码篇方法论强调：**Vite 自身（TS）的逻辑直接断点跟，但 Rolldown/Oxc 这类 Rust 工具链以「架构与边界」视角讲清、给出调试切入点，不强求逐行啃 Rust。** 对绝大多数读者，搞清楚「Vite 喂给 Rolldown 什么、Rolldown 吐回什么」就足够了。

---

## 五、回到源码：Rolldown 在 Vite 8.1.0 里出现的三个地方

亲手在源码里搜 `rolldown`，你会发现它就出现在三个关键位置——这三处恰好对应「dev 预构建 / build 打包 / 配置加载」：

### 5.1 build 打包（[主线 11 构建阶段驱动 Rolldown](../01-Vite实现原理/11-构建阶段驱动Rolldown.md) 已细讲）

文件：`packages/vite/src/node/build.ts`（L892–894）

```ts
const { rolldown } = await import('rolldown')   // 惰性加载,不用 build 不付加载成本
startTime = Date.now()
bundle = await rolldown(rolldownOptions)         // 🔖断点[小册11] Vite 8 build 引擎
```

这里取代了 Vite 7 的 `import('rollup')` + `rollup(rollupOptions)`。watch 模式则走 `build.ts:863` 的 `import('rolldown').watch`。

### 5.2 dev 依赖预构建（原 esbuild 的活）

文件：`packages/vite/src/node/optimizer/index.ts`（L841 附近）

```ts
async function build() {
  const bundle = await rolldown({   // 🔖断点[专题02] dev 预构建也用 Rolldown(Vite 7 时这里是 esbuild)
    // ...把 node_modules 依赖预打包成 .vite/deps 下的少量 ESM
  })
}
```

这是「单引擎收敛」在 dev 侧的体现：预构建从 esbuild 换成了 Rolldown（详见 [主线 04 依赖预构建](../01-Vite实现原理/04-依赖预构建.md)）。

### 5.3 配置文件打包

文件：`packages/vite/src/node/config.ts`（L2435 附近）

```ts
const bundle = await rolldown({   // 把 vite.config.ts 打包成可执行 JS 再加载
  // ...
})
```

`vite.config.ts` 本身要先被打包成 JS 才能在 Node 里执行（详见 [主线 02 配置加载与归一化](../01-Vite实现原理/02-配置加载与归一化.md)），这一步在 Vite 8 也用 Rolldown。

> 把这三处连起来看：**Rolldown 在 Vite 8 里「无处不在」**——build、dev 预构建、配置加载全靠它。这正是「统一打包器」定位的实现证据，也解释了为什么 `rolldown` 是 `package.json` 里少数几个核心 `dependencies` 之一，而 `rollup` 只剩 `devDependencies`。

---

## 六、配置兼容：`rollupOptions` 是怎么「无感」变成 `rolldownOptions` 的

老项目里写的都是 `build.rollupOptions`。Vite 8 没有逼你改名，而是用一个 getter/setter 代理让 `rollupOptions` 实际读写 `rolldownOptions`。

文件：`packages/vite/src/node/utils.ts`（L1258–1292）

```ts
export function setupRollupOptionCompat(buildConfig, path) {
  // 两者都写了就以 rolldownOptions 为准
  buildConfig.rolldownOptions ??= buildConfig.rollupOptions
  // ...
  Object.defineProperty(buildConfig, 'rollupOptions', {
    // 🔖断点[专题02] rollupOptions 读写实际打到 rolldownOptions,老配置无感迁移
    get() { return buildConfig.rolldownOptions },
    set(newValue) { buildConfig.rolldownOptions = newValue },
    configurable: true,
    enumerable: true,
  })
}
```

它在 `config.ts` 里被对 `build`、`worker`、`optimizeDeps`、`ssr.optimizeDeps` 四处分别调用（`config.ts:1408–1415`）。效果是：

- 你写 `build.rollupOptions.output.xxx`，实际写进的是 `rolldownOptions`；
- 读 `config.build.rollupOptions` 拿到的也是同一个对象（`rollupOptions === rolldownOptions`，源码测试 `packages/vite/src/node/__tests__/config.spec.ts` 的 `handles rolldownOptions` 用例用 `expect(mergedConfig.build.rollupOptions).toStrictEqual(...)` 专门验证了这点）；
- 两个都写时，以 `rolldownOptions` 为准。

> 迁移要点：`rollupOptions` 能继续跑，但它只是个「别名」。新项目直接写 `rolldownOptions`；老项目可不动，但要在产物上验证——因为底层换了打包器，**配置名兼容不等于行为完全一致**（尤其 `output.manualChunks` 这类被透传给 Rolldown 解释的字段，分包结果可能有差异；Vite 的原生分包旋钮是 Rolldown 的 `output.codeSplitting`，详见 [主线 11 构建阶段驱动 Rolldown](../01-Vite实现原理/11-构建阶段驱动Rolldown.md)）。

---

## 七、动手对比：同一个项目两套引擎跑一遍

虽然 Vite 8 默认就是 Rolldown，但你可以通过对照「Vite 8 build vs Vite 7 build」直观感受差异（如果手头有 Vite 7 项目）：

```bash
# Vite 8.1.0 源码仓库:看 Rolldown 接到的完整配置
node packages/vite/bin/vite.js build playground/html
# 配合断点 build.ts:894,把 rolldownOptions.input/output/plugins.length 打出来
```

调试建议：在 `build.ts:894` 停下，对比 `rolldownOptions` 与你 `vite.config` 里写的 `rollupOptions`——你会看到 Vite 替你补了大量默认值，且你的 `rollupOptions` 已被「搬进」`rolldownOptions`。这比读任何文档都更直观地说明「兼容层到底做了什么」。

---

## 八、本节小结

**这套实现解决了什么问题？**
Rolldown 用一个「兼容 Rollup API 的 Rust 打包器」同时胜任 dev 预构建、build 打包、配置加载三处，把 Vite 2–7 的双引擎收敛为单引擎，带来 10–30x 的构建提速与 dev/build 行为一致性；`setupRollupOptionCompat` 让海量存量项目的 `rollupOptions` 无感迁移。

**它带来了什么复杂度 / 代价？**
原生二进制带来分发体积、跨平台与调试门槛的上升；「兼容 Rollup API」只覆盖公开钩子契约，依赖 Rollup 内部实现的插件仍可能失效（详见 [专题 04 生态连带影响（实现层）](./专题04-生态连带影响.md)）；`rollupOptions ↔ rolldownOptions` 的代理、`manualChunks` 的透传，都是必须长期维护的兼容成本。

**读完你应当能做到：**

- [ ] 用实现层语言解释 Rollup 与 Rolldown 在定位、速度、语言上的差异，并说清「10–30x」为何是区间而非定值；
- [ ] 在 Vite 8.1.0 源码里指出 Rolldown 出现的三处（`build.ts`、`optimizer/index.ts`、`config.ts`）及各自职责；
- [ ] 说明 `setupRollupOptionCompat` 如何用代理让 `rollupOptions` 变成 `rolldownOptions`，并据此判断老项目升级风险；
- [ ] 理解 `rolldown` 是运行时依赖、`rollup` 只剩 devDependency 这一依赖结构变化。

打包器讲完了，但把 TS/JSX 变成 JS、把产物压缩的，是另一条 Rust 工具链——下一节 [专题 03 Oxc 工具链与边界](./专题03-Oxc工具链与边界.md)。
