# 03 · Vite 8 单打包器解锁的新能力：更灵活的分包 / Module Federation / 持久缓存

> 本节解决的真实工程问题：升级到 Vite 8 不只是「构建变快了」这么简单。当 dev 和 build 都收敛到同一个 Rust 打包器（Rolldown）之后，过去因为「双引擎」而难以实现或实现得别扭的几件事，现在变得自然了。具体就是——**更灵活的分包（**`codeSplitting`**）、Module Federation（微前端）、模块级持久缓存**。本节把这三项逐个落到可运行的 demo 上：你不只是「知道 Vite 8 能干这些」，而是亲手跑出多组分包产物、跑通一个宿主远程加载远程模块的微前端、看到依赖预构建缓存跨进程命中。

事实基线：2026 年中，Vite 8.1.x / Rolldown 1.1.x。Module Federation 通过官方推荐的 `@module-federation/vite` 插件接入（本节实测 1.16.x + Vite 8）。

> 前置：建议先读 [02 各版本升级要点与配置迁移](./02-各版本升级要点与配置迁移.md)，本节的 `codeSplitting` 正是那一节迁移目标的进阶用法。

---

## 一、为什么「单打包器」是这些能力的前提

先把因果讲清楚，否则这三项会显得像「凑数的新功能」。

Vite 5 时代是**双引擎**：dev 用 esbuild 做依赖预构建（不打包业务代码），build 用 Rollup 做生产打包。两套引擎、两套模块图、两套缓存。在这种结构下：

- **分包**只能在 build 侧（Rollup）做，dev 侧根本没有「打包」概念，二者对不齐。
- **Module Federation** 这种「运行时共享模块」的机制，要在 dev 和 build 两套引擎里各实现一遍且行为一致，极其别扭——这也是早年 Vite 做 MF 一直靠社区插件零敲碎打的原因。
- **持久缓存**各管各的：dev 的预构建缓存和 build 的产物缓存是两套体系，难以统一复用。

Vite 8 把 dev 和 build 都收敛到 **Rolldown 一个引擎**之后，上面三件事都变成「在同一套模块图、同一套管线上做一次」：

> **一句话因果：单打包器 = 一套模块图 + 一套管线 + 一套缓存。** 灵活分包、Module Federation、持久缓存，本质都是「在统一管线上才好做的事」。这就是为什么它们是 Vite 8（而不是更早）解锁的。

## 二、能力一：更灵活的分包（codeSplitting）

[02 节](./02-各版本升级要点与配置迁移.md)讲过 `manualChunks → codeSplitting` 的改名，这里讲它**新解锁的灵活度**。

`manualChunks` 是一个回调函数，你只能在里头写一堆 `if (id.includes(...)) return 'xxx'`。`codeSplitting` 是**声明式的多组规则**，每组除了 `name` + `test`，还能配阈值，对分包做精细控制：

```js
build: {
  rolldownOptions: {
    output: {
      codeSplitting: {
        // 全局兜底阈值
        minSize: 0,
        groups: [
          { name: 'vendor-ui',    test: /vendor-ui/ },     // 一组：UI 库
          { name: 'vendor-utils', test: /vendor-utils/ },   // 另一组：工具库
          // 每组还可单独配 minSize / maxSize / minShareCount 等，
          // 控制「太小的别拆、太大的再拆、被引用 N 次以上才单独拆」。
        ],
      },
    },
  },
}
```

它比 `manualChunks` 强在哪：

- **多组规则一目了然**，不用在一个回调里堆 if，团队协作时谁加了哪条规则清清楚楚。
- **阈值控制**：`minSize` 避免拆出一堆几百字节的碎片 chunk；`maxSize` 把超大 chunk 再切开；`minShareCount` 实现「被多个入口共享 ≥N 次才单独拆成 shared chunk」的经典策略。
- **更利于长效缓存**：把「很少变」的依赖单独成组，业务代码改动不影响它的内容哈希，用户浏览器缓存继续命中。

> 配套 demo（见第五节）用两组规则把 `vendor-ui` 和 `vendor-utils` 拆成两个独立 chunk，构建产物里能直接看到 `vendor-ui-*.js` 和 `vendor-utils-*.js`。

## 三、能力二：Module Federation（微前端）

Module Federation（模块联邦）是微前端的核心机制：**多个应用各自独立编译、独立部署，运行时再把彼此的模块「联邦」起来按需远程加载**。一个「宿主（host）」可以在运行时加载「远程（remote）」暴露出来的模块，而宿主构建时**根本没有打包远程的代码**。

它的两个角色：

- **remote（远程应用）**：编译产出一个 `remoteEntry.js` 清单，声明「我对外暴露哪些模块」。
- **host（宿主应用）**：配置里声明「我要消费哪个远程应用、它的 `remoteEntry.js` 在哪」，运行时按需远程加载。

Vite 8 时代官方推荐 `@module-federation/vite` 插件。remote 侧配置：

```js
import { federation } from '@module-federation/vite';

export default defineConfig({
  plugins: [
    federation({
      name: 'remote',
      filename: 'remoteEntry.js',
      exposes: { './widget': './src/widget.js' },  // 对外暴露 widget
      shared: [],
    }),
  ],
  build: { target: 'esnext' },
  server: { origin: 'http://localhost:5174' },
});
```

host 侧配置：

```js
federation({
  name: 'host',
  remotes: {
    remote: {
      type: 'module',
      name: 'remote',
      entry: 'http://localhost:5174/remoteEntry.js',  // 远程清单地址
      entryGlobalName: 'remote',
      shareScope: 'default',
    },
  },
  shared: [],
})
```

host 里这样运行时加载远程模块：

```js
// 'remote' 对应 remotes 键，'widget' 对应 remote 那边 exposes 的 './widget'
const { mount } = await import('remote/widget');
mount(document.querySelector('#app'));
```

关键点：**宿主构建产物里没有** `widget` **的代码**，它是运行时从 `5174` 远程拉回来的。这意味着 remote 团队可以独立发版，宿主无需重新构建就能用上最新的远程模块——这正是微前端「独立部署」的价值。

> 配套 demo（见第五节）提供一对可运行的 host + remote（纯原生 JS，无框架依赖），你能起两个服务、在宿主页面上看到「来自 remote 的 widget」被远程加载进来。

## 四、能力三：模块级持久缓存

Vite 的依赖预构建结果会落盘到 `cacheDir`（默认 `node_modules/.vite/deps`），并按「lockfile + 配置」算一个哈希。**只要哈希没变，下次启动直接复用磁盘上的缓存、跳过重新预构建**——这份缓存跨进程、跨 dev/build 持久存在。

单打包器让这件事更彻底：dev 的预构建与 build 共用 Rolldown 这一套管线，缓存体系得以统一。你可以直观地观察到缓存的「持久」与「命中」：

```bash
# 首次：强制预构建，生成缓存
npx vite optimize --force
#   Optimizing dependencies: mitt   → 真的在预构建

# 再次：哈希一致，直接命中缓存
npx vite optimize
#   Hash is consistent. Skipping. Use --force to override.   → 跳过，命中持久缓存
```

第二条命令输出 `Hash is consistent. Skipping.`——这就是**持久缓存命中**：缓存落在磁盘上，新开一个进程依然能复用，不必重新预构建。这是「同一套缓存管线」带来的开发体验提升。

> 注意区分两种缓存：①「依赖预构建缓存」（`node_modules/.vite/deps`，本节演示的，最直观）；②「构建产物的增量缓存」（Rolldown 在统一管线上持续推进的方向，命中程度取决于版本与改动范围）。本节聚焦 ①，因为它确定存在、可观察。

## 五、动手：三个能力各跑一个 demo

### Demo A：codeSplitting 多组分包

配套代码：[第一阶段-使用篇/05-版本差异与升级迁移/03-单打包器新能力/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/05-版本差异与升级迁移/03-单打包器新能力/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/03-单打包器新能力
npm install
npm run build      # 看产物拆出 vendor-ui 和 vendor-utils 两个独立 chunk
npm run cache:demo # 观察依赖预构建持久缓存的「生成 → 命中」
```

`npm run build` 产物：

```
dist/assets/vendor-utils-B7jV6bvL.js  0.09 kB
dist/assets/vendor-ui-I1btjbwk.js     0.15 kB
dist/assets/index-*.js                ...
```

两组规则各自拆出了独立 chunk——这是 `manualChunks` 也能做、但 `codeSplitting` 写起来更清爽、还能加阈值控制的事。

`npm run cache:demo` 输出（关键两行）：

```
① 首次预构建（vite optimize --force）：Optimizing dependencies: mitt
② 再次预构建（vite optimize）：Hash is consistent. Skipping.
```

第二次「Skipping」就是持久缓存命中的铁证。

### Demo B：Module Federation 宿主远程加载远程模块

配套代码：[第一阶段-使用篇/05-版本差异与升级迁移/03b-模块联邦/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/05-版本差异与升级迁移/03b-模块联邦/)（含 `remote/` 和 `host/` 两个子项目）

```bash
# 终端 1：构建并预览 remote（占用 5174）
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/03b-模块联邦/remote
npm install && npm run build && npm run preview

# 终端 2：构建并预览 host（占用 5173）
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/03b-模块联邦/host
npm install && npm run build && npm run preview
```

然后浏览器打开 `http://localhost:5173/`（host），你会看到页面上出现一块「👋 我是来自 remote 的 widget」——它的代码不在 host 的产物里，是运行时从 `5174` 的 `remoteEntry.js` 远程加载进来的。

观察点：

1. 打开 `remote/dist/`，能看到 `remoteEntry.js`——这是 remote 对外的「联邦清单入口」。
2. host 构建日志里没有 widget 的业务代码体积；它只在运行时通过 `import('remote/widget')` 远程拉取。
3. 改一下 `remote/src/widget.js` 的文案，只重新构建/预览 remote（不碰 host），刷新 host 页面就能看到变化——这就是「remote 独立发版、host 无需重建」。

### Demo C：持久缓存

即上面 Demo A 的 `npm run cache:demo`，单独再强调一次它的价值——这是「为什么 Vite 第二次启动比第一次快得多」的底层原因。

## 六、什么时候真的用得上（别为新而新）

- **codeSplitting**：几乎所有有第三方依赖的项目都该配，做长效缓存。**推荐默认就用**。
- **Module Federation**：只有**真正做微前端 / 多团队独立部署**时才需要。普通单体应用用它纯属增加复杂度——别因为「Vite 8 支持了」就上 MF。
- **持久缓存**：自动生效，你不用配，知道它存在、知道清缓存（删 `node_modules/.vite`）能解决「预构建结果脏了」的诡异问题即可。

## 七、本节小结

- 这三项能力的共同前提是「单打包器」：dev/build 收敛到 Rolldown 一套模块图 + 一套管线 + 一套缓存后，它们才好做。
- **codeSplitting**：从 `manualChunks` 的函数回调升级为声明式多组规则，支持 `minSize`/`maxSize`/`minShareCount` 等阈值，更易读、更利于长效缓存。
- **Module Federation**：用 `@module-federation/vite`，host 运行时远程加载 remote 暴露的模块，宿主构建不打包远程代码，实现微前端的独立部署。
- **持久缓存**：依赖预构建结果按哈希落盘到 `node_modules/.vite`，跨进程命中（`Hash is consistent. Skipping.`）。
- 选型克制：codeSplitting 普遍适用；MF 只用于真微前端；持久缓存自动生效、了解清缓存方法即可。

## 八、可直接用于项目的 checklist

- [ ] 有第三方依赖就配 `codeSplitting.groups`，把稳定依赖单独成组做长效缓存。
- [ ] 用 `minSize` 防碎片、`maxSize` 切超大 chunk、`minShareCount` 抽公共 chunk。
- [ ] 只有「多团队独立部署 / 微前端」需求才上 Module Federation，否则不要引入。
- [ ] 上 MF 时确认 remote 的 `remoteEntry.js` 地址、host 的 `remotes` 配置、共享依赖 `shared` 三者对齐。
- [ ] 遇到「预构建结果像是脏了」的怪问题，先删 `node_modules/.vite` 重来。
- [ ] 任何「新能力」先问「我的项目真需要吗」，别为新而新地堆复杂度。