# 02 · Rollup 基础：插件钩子模型与 rollup / Vite 的关系

> 本节解决的真实工程问题：你大概率遇到过这些困惑——为什么 Vite 文档说「Vite 插件是 Rollup 插件的超集」？为什么一个 `rollup-plugin-xxx` 能直接塞进 Vite 的 `plugins` 用？为什么有的 Rollup 插件在 Vite **dev 下不生效、build 下才生效**？为什么 Vite 8 把 `build.rollupOptions` 改叫 `rolldownOptions`、有些插件就「兼容性失效」了？这些问题的答案，全藏在「Rollup 的钩子模型」以及「Vite 在 dev / build 两个阶段分别由谁来跑这些钩子」里。本节把这套关系讲透，让你不仅会用插件，还知道它「为什么这样工作」——这也是衔接第二阶段源码的关键基础。

事实基线：2026 年中，Vite 8.x（build 默认 Rolldown，Rolldown 1.0 已于 2026.05 稳定）。

---

## 一、为什么学 Vite 插件，要先懂 Rollup

一句话：**Vite 的插件 API 直接复用了 Rollup 的插件 API**。[01 节](./01-插件开发从最小插件到完整生命周期.md) 提到钩子分两类，其中 `resolveId / load / transform / buildStart / renderChunk / generateBundle` 这些「通用钩子」，名字、参数、语义全部来自 Rollup。Vite 只是额外加了几个自己的钩子（`config`、`configureServer`、`transformIndexHtml`、`hotUpdate`）。

所以官方那句「**Vite 插件是 Rollup 插件的超集**」可以这样理解：

```
Vite 插件接口  =  Rollup 插件的全部钩子  +  Vite 特有的几个钩子
```

这意味着：一个只用了 Rollup 通用钩子的插件，**天然就是合法的 Vite 插件**，反过来不一定。要看懂、写好、排查 Vite 插件，绕不开 Rollup 这套钩子模型。

## 二、Rollup 的构建模型：两个阶段

Rollup 把「从源码到产物」分成清清楚楚的**两个阶段**，钩子也据此分两组。这张分阶段的图是理解一切的地基：

```
┌─────────────────── Build 阶段（构建模块图）───────────────────┐
│  options → buildStart                                          │
│    对每个模块循环：resolveId → load → transform                │
│    （解析依赖，继续递归，直到整张模块图建完）                   │
│  buildEnd                                                       │
└────────────────────────────────────────────────────────────────┘
                          │  模块图建好了
                          ▼
┌─────────────────── Output 阶段（生成产物）────────────────────┐
│  outputOptions → renderStart                                   │
│    renderChunk（对每个 chunk 改最终代码）                       │
│    generateBundle（拿到全部产物，可增删文件）                   │
│  writeBundle（产物已写到磁盘后）                                │
└────────────────────────────────────────────────────────────────┘
```

- **Build 阶段**：回答「有哪些模块、每个模块内容是什么」。核心三件套 `resolveId`（这个 import 指向谁）→ `load`（它的内容是什么）→ `transform`（把内容改写一下）。
- **Output 阶段**：回答「这些模块怎么拼成最终的几个文件」。`renderChunk` 改每个 chunk 的最终代码，`generateBundle` 拿到全部产物清单（可以再加文件，比如生成一个 `manifest.json`），`writeBundle` 在文件落盘后回调。

> 把 [01 节](./01-插件开发从最小插件到完整生命周期.md) 的「dev 独有 / build 独有」对上：**Output 阶段的钩子只在 build 跑**，因为只有 build 才产出 bundle。dev 没有 Output 阶段。

## 三、钩子类型：不只是「函数」，还有调用约定

Rollup 的每个钩子还有「类型」，决定多个插件都挂了同一钩子时怎么协作。常见三种：

| 类型 | 含义 | 典型钩子 |
|---|---|---|
| **first** | 依次调用各插件，**谁第一个返回非 null 就用谁的**，后面不再调 | `resolveId`、`load` |
| **sequential** | 按顺序串行，前一个的输出喂给后一个 | `transform`（每个插件依次改同一份代码） |
| **parallel** | 各插件并行执行，互不依赖 | `buildStart`、`buildEnd` |

这解释了很多行为：

- 为什么两个插件都写了 `resolveId`，只有一个生效？因为 `resolveId` 是 **first** 型——第一个返回结果的就「赢了」，所以 `enforce: 'pre'` 的插件能抢先解析。
- 为什么 `transform` 可以多个插件「接力」改同一份代码？因为它是 **sequential** 型——前一个插件的输出是后一个的输入。框架插件编译 `.vue` 之后，你的插件还能继续 transform 它的产物，就是靠这个。

> 理解钩子类型 + `enforce` 顺序，你就能预测「多个插件同时挂一个钩子」时的最终结果，而不是靠试。

## 四、插件上下文：钩子里不只返回值，还能调用 Rollup 能力

Rollup 插件钩子除了接收参数、返回结果，还能通过 `this` 调用一组由 Rollup/Rolldown 提供的方法。这个 `this` 不是随便绑定的 JS 对象，而是**插件上下文（Plugin Context）**。它让插件可以主动和构建器交互，而不只是被动处理代码。

常见方法有这些：

- **`this.emitFile(...)`**：往最终产物里额外发文件。比如生成一个 `manifest.json`、复制一份静态资源，或者额外产出一个 chunk。
- **`this.getFileName(referenceId)`**：拿到 `emitFile` 返回引用对应的最终文件名。因为文件名可能带 hash，不能提前写死。
- **`this.addWatchFile(file)`**：让 Rollup/Vite 额外监听某个文件。插件读取了配置文件、模板文件时常用，否则这些文件变了构建器可能不知道。
- **`this.resolve(source, importer)`**：复用 Rollup 的解析能力，问构建器「这个 import 最终会解析到哪里」。
- **`this.load({ id })`**：让构建器加载某个模块，复用其它插件的 `load` 结果。
- **`this.getModuleInfo(id)` / `this.getModuleIds()`**：读取模块图信息，适合做依赖分析、产物统计。
- **`this.warn(...)` / `this.error(...)`**：用构建器统一格式输出 warning 或中断构建。

例如在 Output 阶段额外生成一个文件：

```js
function manifestPlugin() {
  return {
    name: 'manifest-plugin',
    generateBundle() {
      const ref = this.emitFile({
        type: 'asset',
        fileName: 'build-manifest.json',
        source: JSON.stringify({ generatedAt: Date.now() }, null, 2),
      });

      this.warn(`generated ${this.getFileName(ref)}`);
    },
  };
}
```

注意两点：

1. 要用 `this` 的钩子不要写成箭头函数，否则拿不到插件上下文。
2. `emitFile` 这类和最终产物相关的方法，主要在 build 的 Output 阶段有意义；dev 不产出 bundle，自然也没有真正的最终文件清单。

## 五、rollup 与 Vite 的关系：dev 和 build 是「两套人马跑同一份插件」

这是本节最重要的结论。同一份 `plugins`，在 dev 和 build 两个阶段，**跑钩子的「执行者」根本不是同一个**：

### dev 阶段：Vite 用「插件容器」模拟 Rollup

dev 时没有 Rollup 参与打包（Vite dev 不打包，按需编译，见 [01 核心使用 / 依赖预构建](../01-核心使用/05-依赖预构建.md)）。但你的插件里写了 `resolveId / load / transform` 啊——谁来调用它们？

答案是 Vite 内部一个叫 **PluginContainer（插件容器）** 的东西。它**模拟了 Rollup 的构建钩子调用约定**：每当浏览器请求一个模块，Vite 就用插件容器依次跑一遍 `resolveId → load → transform`，效果和 Rollup 在 build 时跑这些钩子一致。

```
dev：浏览器请求 /src/App.jsx
   → Vite PluginContainer（模拟 Rollup）
   → 跑你插件的 resolveId / load / transform
   → 返回编译后的单个模块
```

所以：**只用了 `resolveId / load / transform` 的插件，dev 下会被插件容器调用，行为和 build 一致**。这也是 Vite「dev/build 同源」的实现基础（插件容器的细节是第二阶段源码篇的重头戏，这里先建立概念）。

### build 阶段：直接交给真正的打包器

build 时 Vite 不再「模拟」，而是把你的 `plugins` **原样交给真正的打包器**去跑完整的两阶段流程（Build + Output）：

- **Vite 2 ~ 7**：build 用的是 **Rollup**（JavaScript 实现）。
- **Vite 8（事实基线）**：build 默认用 **Rolldown**（Rust 实现），并兼容 Rollup 的插件钩子 API。

```
build：vite build
   → 把 plugins 交给 Rolldown（Vite 8）/ Rollup（Vite 7 及以前）
   → 完整跑 Build 阶段 + Output 阶段（含 renderChunk/generateBundle/writeBundle）
```

### 一句话总结这层关系

```
              dev                          build
plugins ──►  Vite PluginContainer     │   Rolldown（Vite 8）/ Rollup（≤7）
            （只模拟 build 钩子）       │   （完整 build + output 钩子）
```

这张图直接解释了开头那些困惑：

- **「为什么有的 Rollup 插件 dev 不生效、build 才生效？」** —— 因为它用的是 **Output 阶段钩子（`renderChunk`/`generateBundle`）**，dev 的插件容器根本不跑 Output 阶段，自然只有 build 时（交给 Rollup/Rolldown）才触发。
- **「为什么有些 esbuild 风格的插件（`onResolve`/`onLoad`）升级后不生效？」** —— `onResolve` / `onLoad` 是 esbuild 的插件 API，不属于 Rollup 钩子模型。它能不能生效，关键看当前这段流程有没有真正进入 esbuild，并且有没有把 esbuild 的插件配置传进去：依赖预构建、JS/TS 转换、压缩，或者某个 Rollup/Rolldown 插件内部再调用 esbuild，都可能让 esbuild 参与；但如果升级后这段具体子流程不再调用 esbuild，或只是调用 esbuild 的 transform/minify 而没有走 esbuild plugin API，原来的 `onResolve` / `onLoad` 插件就不会被触发。根因不是“Vite 版本号”，而是“这条子链路是否还走 esbuild 的插件 API”（详见第二阶段「生态连带影响」）。

## 六、Vite 8 / Rolldown 带来的变化（使用者要知道的部分）

Vite 8 把 build 打包器从 Rollup 换成 Rust 写的 Rolldown，对**插件使用者**主要有几处可感知变化：

1. **配置入口改名**：`build.rollupOptions` → `build.rolldownOptions`（Vite 8 保留兼容层，旧名仍能用一段时间，但建议迁移）。`output.manualChunks` → `output.codeSplitting`（分包配置，早期过渡名 `advancedChunks` 已弃，见 [05 章](../05-版本差异与升级迁移/02-各版本升级要点与配置迁移.md)）。
2. **绝大多数 Rollup 插件仍可用**：Rolldown 兼容 Rollup 的插件钩子 API，常规插件（只用标准钩子的）直接复用。
3. **依赖 Rollup「内部实现」的插件可能失效**：少数插件 hack 了 Rollup 的内部数据结构 / 私有 API，Rolldown 内部是 Rust 实现，这类会出问题——这也是 `rolldown-vite` 作为迁移隔离层存在的原因（升级专题见 [05 版本差异与升级迁移](../05-版本差异与升级迁移/README.md)）。
4. **钩子模型本身没变**：`resolveId / load / transform / renderChunk / generateBundle` 这套语义在 Rolldown 里保持一致，所以你**学的钩子模型不会过时**。

> 关键认知：从 Rollup 到 Rolldown，是「**换了引擎、保留了插件接口**」。你掌握的钩子模型是可迁移资产；变的主要是配置字段名和极少数依赖内部实现的插件。

## 七、动手：一个纯 Rollup 钩子插件，观察 dev / build 行为差异

配套 demo 写了一个**只用 Rollup 通用钩子**的插件，在每个钩子里打印日志。你分别跑 dev 和 build，对照终端输出，亲眼看到「哪些钩子两边都跑、哪些只 build 跑」。

配套代码：[第一阶段-使用篇/03-插件与生态/02-Rollup钩子模型/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/03-插件与生态/02-Rollup钩子模型/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/03-插件与生态/02-Rollup钩子模型
npm install
npm run build    # 看完整两阶段钩子：buildStart→resolveId/load/transform→renderChunk→generateBundle→writeBundle
npm run dev      # 看 dev：只有 buildStart + 按请求触发的 resolveId/load/transform，没有 Output 阶段钩子
```

观察点：

1. `build` 日志里能看到完整顺序：`buildStart` → 多次 `resolveId/load/transform` → `renderChunk` → `generateBundle`（这里插件还顺手生成了一个 `build-manifest.json`）→ `writeBundle`。
2. `dev` 日志里**只有** `buildStart` 和按浏览器请求触发的 `resolveId/load/transform`——`renderChunk/generateBundle/writeBundle` 一次都不出现。这就是「dev 没有 Output 阶段」的实证。
3. demo 里同一个插件不加任何改动，在 dev（插件容器模拟）和 build（Rolldown）下都能跑——验证「Vite 插件 = Rollup 插件超集」。
4. 看 `generateBundle` 里 `this.emitFile` 生成的额外产物，体会 Output 阶段「能增删最终文件」的能力（dev 做不到）。

## 八、本节小结

- Vite 插件 API 复用 Rollup 插件 API：**Vite 插件 = Rollup 全部钩子 + Vite 特有钩子（超集）**。
- Rollup 分两阶段：**Build 阶段**（`resolveId→load→transform` 建模块图）和 **Output 阶段**（`renderChunk/generateBundle/writeBundle` 生成产物）。
- 钩子有类型：`resolveId/load` 是 first（谁先返回用谁），`transform` 是 sequential（接力改代码），`buildStart` 是 parallel。配合 `enforce` 决定最终行为。
- Rollup 钩子里还能通过插件上下文调用 `this.emitFile`、`this.addWatchFile`、`this.resolve`、`this.warn` 等方法，主动生成文件、监听额外文件或复用解析能力。
- dev 与 build 是「两套执行者跑同一份插件」：dev 由 Vite 的 **PluginContainer 模拟** Rollup 的 Build 钩子（无 Output 阶段）；build 把插件交给真正的打包器（Vite 8 是 **Rolldown**）完整跑两阶段。
- 这解释了「Output 钩子 dev 不触发」「依赖 Rollup 内部实现的插件在 Rolldown 失效」等现象。
- Rollup → Rolldown 是「换引擎、留接口」：钩子模型稳定，主要变化是 `rollupOptions→rolldownOptions`、`manualChunks→codeSplitting` 等配置名（`advancedChunks` 只是早期 Rolldown 过渡名，已 deprecated）。

## 九、可直接用于项目的 checklist

- [ ] 判断一个钩子会不会在 dev 触发：属于 Build 阶段（resolveId/load/transform）→ 两边都触发；属于 Output 阶段（renderChunk/generateBundle/writeBundle）→ 只 build 触发。
- [ ] 装第三方 Rollup 插件前，先看它用的钩子属于哪个阶段，预判它在 dev 是否生效。
- [ ] 多个插件挂同一钩子，结果不符预期时，回忆钩子类型（first/sequential/parallel）+ `enforce` 顺序去解释。
- [ ] 钩子里要用 `this.emitFile`、`this.resolve` 等插件上下文方法时，不要写箭头函数；需要额外产物时优先放在 `generateBundle` 这类 Output 阶段钩子里。
- [ ] 从 Vite 7 升级到 8 时，把 `build.rollupOptions` 迁到 `rolldownOptions`、`manualChunks` 迁到 `codeSplitting`。
- [ ] 遇到「升级到 Vite 8 后某插件报错/失效」，优先怀疑它依赖了 Rollup 内部实现，查其是否有 Rolldown 兼容版本或用 `rolldown-vite` 过渡。
- [ ] 用 esbuild 风格 `onResolve/onLoad` 的插件不能当 Rollup 插件用，别混淆两套钩子模型。
