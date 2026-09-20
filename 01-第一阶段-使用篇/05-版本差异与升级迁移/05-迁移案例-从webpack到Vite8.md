# 05 · 迁移案例：从 webpack / CRA / Vue CLI / 老 Vite 迁到 Vite 8

> 本节解决的真实工程问题：上一节主讲「Vite 5 升 Vite 8」（同一工具升版本），本节主讲**换工具**——把一个 webpack / Create React App / Vue CLI 老项目搬到 Vite 8，并补一份「老 Vite 项目」的分流 checklist。这两类迁移难度完全不同：升版本主要改配置名，换工具要面对「webpack 专有 API 在 Vite 里根本不存在」的硬迁移点，比如 `require.context`、`process.env.REACT_APP_`*、`require()` 同步加载、`DefinePlugin`。本节按「迁移分流 → 配置映射 → 代码改写 → 构建差异 → 回滚」拆解，并用一个可运行 demo 把两个最高频、最容易翻车的改写点（`require.context → import.meta.glob`、`process.env → import.meta.env`）跑给你看。

事实基线：2026 年中，目标 Vite 8.1.x。各源工具（webpack 5 / CRA 5 / Vue CLI 5）的概念以其经典形态为准。

> 前置：建议先读 [01 版本全景](./01-两条演进主线与版本全景.md) 了解 Vite 8 的能力边界。本节不教 Vite 基础用法（见 [01 核心使用](../01-核心使用/README.md)），只聚焦「从别的工具搬过来」的差异。

---

## 一、先分流：你是在升 Vite，还是在换工具

迁移前先把项目分成两类，路线完全不同：


| 来源项目                    | 迁移性质   | 主要工作                      | 重点章节                                   |
| ----------------------- | ------ | ------------------------- | -------------------------------------- |
| Vite 3/4/5/6/7          | 同工具升版本 | 升 Node、处理废弃配置、验证插件兼容、对比产物 | [04 企业升级实战](./04-企业升级实战-从Vite5渐进升级.md) |
| webpack / CRA / Vue CLI | 换工具    | 改配置，也要改写 webpack 私有模块 API | 本节                                     |


老 Vite 项目不是“换工具”，不要套 webpack 迁移思路。建议按这条短路径走：先把 Node 升到 20.19+ / 22.12+，再直接上 Vite 8 跑构建收集告警，把 `rollupOptions → rolldownOptions`、`manualChunks → codeSplitting` 等逐项消掉，最后做插件兼容和产物 diff。大型项目可先用 `rolldown-vite` 隔离打包器风险，详见 [04 节](./04-企业升级实战-从Vite5渐进升级.md)。

## 二、迁移心智：从「配置驱动」到「约定 + ESM 原生」

webpack/CRA/Vue CLI 的世界观是**配置驱动**：入口、出口、loader、plugin，什么都在配置里显式声明，且大量依赖 webpack 自创的模块系统（`require.context`、`require.ensure`、magic comments……）。

Vite 的世界观是**约定优于配置 + 拥抱 ESM 原生**：以 `index.html` 为入口、约定 `src/` 结构、用浏览器原生 ESM。所以迁移不是「把 webpack 配置翻译成 Vite 配置」那么简单——**有些 webpack 专有 API 在 Vite 里压根没有对应配置，必须改写源代码**。

> 一句话抓住迁移本质：**配置层多数能找到对应项（平移），但凡是 webpack 私有的模块魔法，都要在源码层改写成 ESM 原生或 Vite 等价物。** 后者才是迁移真正费时、真正会翻车的地方。

## 三、配置层映射表（平移为主）

配置层的迁移大多是「找对应字段」，对照改即可：


| webpack / CRA / Vue CLI                | Vite 8                                       | 说明                                                                                                  |
| -------------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `entry` / `output`                     | 以 `index.html` 为入口，约定输出                      | 多数项目无需显式配                                                                                           |
| `output.publicPath` / CRA `PUBLIC_URL` | `base`                                       | 子路径部署用                                                                                              |
| `resolve.alias`                        | `resolve.alias`                              | 写法几乎一致                                                                                              |
| `resolve.extensions`                   | `resolve.extensions`                         | 一致                                                                                                  |
| `devServer.proxy`                      | `server.proxy`                               | 一致                                                                                                  |
| `DefinePlugin` 注入环境变量                  | `import.meta.env` + `.env` 文件                | 见第四节，**有安全约定差异**                                                                                    |
| `module.rules`（loader）                 | 内置 + 插件                                      | CSS/资源 Vite 内置；特殊 loader 找对应插件                                                                      |
| `splitChunks`                          | `build.rolldownOptions.output.codeSplitting` | 见 [03 节](./03-Vite8单打包器解锁的新能力.md)                                                                   |
| Vue CLI 整体                             | `@vitejs/plugin-vue`                         | 见 [02 章 Vue 接入](../02-框架集成与测试/03-Vue与Svelte接入.md)                                                   |
| CRA `react-scripts`                    | `@vitejs/plugin-react`                       | Vite 8 下主包 v6 已接入 Oxc Fast Refresh，见 [02 章 React 接入](../02-框架集成与测试/02-React接入与Oxc接管ReactRefresh.md) |


这部分照表改就行，不展开。真正的难点在下一节。

## 四、代码层硬迁移点（必须改写源码）

### 硬迁移点 1：`require.context` → `import.meta.glob`

webpack 里「自动批量引入一个目录下所有模块」是个超高频用法（路由表、自动注册组件、国际化文件……）：

```js
// webpack 专有 API —— Vite 里根本不存在，照搬必报错
const ctx = require.context('./pages', false, /\.js$/);
ctx.keys().forEach((key) => {
  const mod = ctx(key);
  registerRoute(mod.route, mod.render);
});
```

Vite 的等价物是**编译期**的 `import.meta.glob`：

```js
// Vite 等价写法：编译期把匹配的模块收集成一个对象
const pageModules = import.meta.glob('./pages/*.js', { eager: true });
// eager: true → 直接拿到模块对象（相当于一组静态 import）
// 不加 eager → 每个值是返回 Promise 的懒加载函数，天然支持路由级代码分割

for (const [file, mod] of Object.entries(pageModules)) {
  registerRoute(mod.route, mod.render);
}
```

两个关键差异要记牢：

- `import.meta.glob` **是编译期静态分析的**——glob 模式必须是字面量字符串，不能拼接变量。这换来了 tree-shaking 友好和懒加载能力。
- `eager` **开关**对应两种场景：要立即用就 `eager: true`；要按路由懒加载就不加 `eager`（拿到的是 `() => import(...)`）。

### 硬迁移点 2：`process.env.REACT_APP_*` → `import.meta.env.VITE_*`

环境变量是迁移时最容易踩的**安全坑**：

```js
// CRA / webpack：process.env + REACT_APP_ 前缀
const title = process.env.REACT_APP_TITLE;

// Vite：import.meta.env + VITE_ 前缀
const title = import.meta.env.VITE_APP_TITLE;
```

不只是改个前缀名，背后有**安全语义差异**：

- Vite 只把 `VITE_` **前缀**的变量暴露到客户端代码，其余 `.env` 变量留在服务端/构建期不外泄。这是 Vite 防止「误把密钥打进前端包」的刻意设计。
- 所以迁移时要审一遍：原来 `REACT_APP_*` 里有没有**本不该暴露给浏览器**的东西（密钥、内部地址）？借迁移的机会把它们从客户端变量里清出去。

### 其他常见硬迁移点（速查）

- `require('x')`（CommonJS 同步导入）→ `import x from 'x'`（ESM）。Vite 是 ESM 优先，源码里的 CJS 要改成 ESM。
- webpack magic comments（`/* webpackChunkName */`）→ Rolldown 的 `codeSplitting` 命名规则（见 [03 节](./03-Vite8单打包器解锁的新能力.md)）。
- `require.ensure` / `import(/* ... */)` 懒加载 → 标准动态 `import()`（Vite 原生支持）。
- 静态资源 `require('./img.png')` → `import imgUrl from './img.png'` 或放 `public/`（见 [04 静态资源](../01-核心使用/04-静态资源CSS别名与路径解析.md)）。

## 五、构建差异：迁移后该验收什么

换工具后，**产物结构和行为会变**，不能只看「能跑起来」。重点对比：

- **分包结构不同**：webpack 的 `splitChunks` 与 Rolldown 的 `codeSplitting` 策略不同，chunk 数量和划分会变。用产物体积报告对比，确认没有「该拆的没拆、首屏包暴涨」。
- **dev 行为差异**：webpack dev 是「打包后再服务」，Vite dev 是「按需 ESM、不打包业务代码」。首屏依赖多时 Vite 首次启动要做依赖预构建（见 [05 依赖预构建](../01-核心使用/05-依赖预构建.md)），别误判为「卡住」。
- **环境变量行为**：上一节的 `VITE_` 前缀约定会让一部分原 `REACT_APP_`* 变量「在前端读不到了」——这往往是 feature 不是 bug，但要在迁移验收时确认。
- **CSS / 资源处理**：CRA/Vue CLI 内置的一些 loader 行为（CSS Modules 命名、资源内联阈值）默认值可能和 Vite 不同，逐项核对。

## 六、回滚方案：换工具比升版本更要留后路

换工具是「大手术」，回滚预案必须在动手前就备好：

- **并行而非替换**：迁移期保留老的 webpack/CRA 构建脚本（`build:webpack`），新加 Vite 脚本（`build:vite`），两套并存。验证 Vite 产物没问题前，**线上仍走老链路**。
- **灰度切换**：先用 Vite 产物在预发/灰度环境跑，对比关键页面、核心指标，再逐步切流量。
- **保留产物**：上线前保留上一版（webpack）的可部署产物，出问题直接回滚部署，而不是回滚代码重新构建。
- **分模块迁移**（大型项目）：能拆就别整体一次性换。先迁一个独立子应用/页面，跑稳了再推广。

> 和 [04 节](./04-企业升级实战-从Vite5渐进升级.md) 一个道理：**「出问题多久能回到已知可用状态」决定了你的步子能迈多大。** 换工具的回滚点要比升版本留得更厚。

## 七、动手：跑通两个最高频硬迁移点

配套 demo 把「`require.context → import.meta.glob`」和「`process.env → import.meta.env`」两个改写点做成一个可运行的最小项目（纯原生 JS，模拟一个 CRA 风格的多页面自动注册）。

配套代码：[第一阶段-使用篇/05-版本差异与升级迁移/05-迁移案例/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/05-版本差异与升级迁移/05-迁移案例/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/05-迁移案例
npm install
npm run dev      # 页面自动列出 src/pages 下被 import.meta.glob 收集到的所有页面
npm run build
```

观察点：

1. `src/main.js` 顶部的注释先给出 webpack 的 `require.context` 原写法，紧接着是 Vite 的 `import.meta.glob` 改写——对照着看就是一份迁移 diff。
2. 在 `src/pages/` 下**新建一个** `xxx.js`（导出 `route` 和 `render`），不改任何注册代码，刷新页面——新页面自动出现。这证明 `import.meta.glob` 复刻了 `require.context` 的「目录自动收集」能力。
3. 页面标题来自 `.env` 里的 `VITE_APP_TITLE`（对应 CRA 的 `REACT_APP_TITLE`）。试着把 `.env` 里的变量名去掉 `VITE_` 前缀，重启 dev——标题读不到了，亲手验证「只有 `VITE_` 前缀才暴露到客户端」的安全约定。

## 八、本节小结

- 升版本（Vite 5→8）和换工具（webpack/CRA→Vite）是两回事：前者改配置名，后者要改写源码里 webpack 专有的模块魔法。
- 配置层多数能平移（`alias`/`proxy`/`base`/`splitChunks→codeSplitting`），照映射表改即可。
- 真正费时、易翻车的是代码层硬迁移点：`require.context→import.meta.glob`、`process.env.REACT_APP_*→import.meta.env.VITE_*`、CJS→ESM、magic comments→codeSplitting。
- `import.meta.glob` 是编译期静态分析（glob 必须字面量），用 `eager` 切换「立即引入 / 懒加载」。
- `VITE_` 前缀约定有安全语义：借迁移清理掉本不该暴露给前端的变量。
- 换工具比升版本更要留后路：两套构建并存、灰度切换、保留可回滚产物、大项目分模块迁移。

## 九、可直接用于项目的 checklist

- [ ] 先按配置映射表把 `alias`/`proxy`/`base`/分包等配置平移到 `vite.config`。
- [ ] 全局搜 `require.context`，逐个改写成 `import.meta.glob`（注意 glob 必须是字面量、按需选 `eager`）。
- [ ] 全局搜 `process.env.`，把客户端用到的改成 `import.meta.env.VITE_*`，并审查有无敏感变量被暴露。
- [ ] 全局搜源码里的 `require(`，改成 ESM `import`。
- [ ] 把 webpack magic comments / `require.ensure` 改成标准动态 `import()` + `codeSplitting` 命名。
- [ ] 迁移后用「产物体积/分包 diff + 关键页面回归」验收，别只看「能启动」。
- [ ] 保留老构建链路与可回滚产物，先灰度、再切流量；大项目分模块迁移。