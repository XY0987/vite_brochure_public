# 03 · SSR 工程坑位排查手册：水合不一致 / 外部化 / 条件导入 / 环境变量泄露

> 本节解决的真实工程问题：SSR（服务端渲染）一上手就是一连串「看起来玄学」的报错和警告——浏览器控制台疯狂刷 `Hydration mismatch` / `Text content did not match`，页面闪一下又变回去；服务端启动直接 `ERR_REQUIRE_ESM` 或 `Unexpected token 'export'` 崩掉；代码里写了 `window` / `document`，dev 一切正常、SSR 一跑就 `window is not defined`；最吓人的是——**你的某个密钥莫名其妙出现在了浏览器能看到的 HTML 里**。这些不是框架 bug，而是 SSR 这套「同一份代码要在 Node 和浏览器两个世界各跑一遍」的模型必然带来的工程坑位。本节把四个最高频的坑——**水合不一致、服务端外部化、条件导入、环境变量泄露**——逐个拆开：现象长什么样、根因是什么、怎么修、怎么提前避免。

事实基线：2026 年中，Vite 8.x。SSR 机制（`ssrLoadModule`、`ssr.*` 配置、`import.meta.env.SSR`）在 Vite 各版本相对稳定。

> 前置：本节假设你已了解 SSR 的基本流程（[01 核心使用 / SSR](../01-核心使用/06-SSR库模式多页应用.md) 里的最小 SSR：Express + Vite 中间件 + `ssrLoadModule`）。这里不重复搭建，专攻「坑位排查」。

---



## 一、SSR 的根本矛盾：同一份代码，两个世界各跑一次

理解所有坑之前，先记住这张图：

```
        服务端（Node）                     浏览器
源码 ──> 执行一次 ──> 产出 HTML 字符串 ──> 发给浏览器 ──> 再执行一次（hydration 水合）
        有 process            ↑ 同一份组件代码          有 window/document
        没有 window           └────────────────────────┘ 没有 process
```

**同一份组件代码，先在 Node 跑一遍生成 HTML，再在浏览器跑一遍「认领」这段 HTML（水合）。** 四大坑全部源于这个事实：

- 两次执行**结果不一致** → 水合不一致（坑一）。
- 代码用了**只有某一边才有的 API** → 条件导入/`window is not defined`（坑三）。
- 服务端打包时**依赖该不该打进包** → 外部化（坑二）。
- **服务端的数据/密钥**怎么安全传给浏览器、哪些绝不能传 → 环境变量泄露（坑四）。

逐个看。

## 二、坑一：水合不一致（Hydration mismatch）

### 现象

浏览器控制台报 `Hydration failed` / `Text content does not match server-rendered HTML`，页面可能闪烁（先显示服务端版本，水合后被客户端版本覆盖）。

### 根因

**服务端生成 HTML 时和客户端水合时，渲染出的内容不一样。** 最常见的几个源头：

```js
// ❌ 三种典型的「两次结果必然不同」
const time = new Date().toISOString();      // 服务端渲染时刻 ≠ 客户端水合时刻
const id = Math.random();                   // 两次随机数不同
if (typeof window !== 'undefined') { ... }  // 服务端走 else、客户端走 if，结构不同
```

服务端在 T1 时刻算出时间写进 HTML，浏览器在 T1+几百毫秒 后水合、又算了一次，两个时间字符串对不上——mismatch。

### 修法：服务端算好，随状态下发，客户端复用

核心原则：**凡是「服务端和客户端都要用、但又会变」的值，由服务端算一次，序列化进 HTML，客户端直接读，不要各算各的。**

```js
// 服务端：算好数据，连同 HTML 一起返回
export async function render() {
  const state = { renderedAt: new Date().toISOString() };
  const html = `<span>${state.renderedAt}</span>`;
  return { html, state };
}
```

```html
<!-- 服务端把 state 序列化进 HTML -->
<script>window.__SSR_STATE__ = {"renderedAt":"2026-06-27T03:31:29.587Z"};</script>
```

```js
// 客户端：读服务端下发的 state，而不是自己重新 new Date()
const state = window.__SSR_STATE__;
render(state);   // 用同一份 renderedAt → 与服务端 HTML 完全一致
```

> 这就是所有 SSR 框架里 `window.__INITIAL_STATE__` / `__NUXT__` / `__remixContext` 的由来——它们都是「服务端把状态序列化下发、客户端复用」这一招的工程化封装。



## 三、坑二：服务端外部化（external / noExternal）

### 现象

服务端启动或渲染时报：

```
Error [ERR_REQUIRE_ESM]: require() of ES Module ...
# 或
SyntaxError: Unexpected token 'export'
```

### 根因

SSR 构建时，Vite 对依赖有两种处理：

- **外部化（external）**：不把依赖打进 SSR 包，运行时由 Node 直接加载。`node_modules` 里的依赖**默认外部化**（省去打包、贴近 Node 真实行为）。
- **打进包（noExternal）**：把依赖编译、打进 SSR 包。

矛盾点：当一个依赖是 **ESM-only**、或它内部用了**需要 Vite 转换的东西**（`import './x.css'`、别名、`import.meta` 等），它被「外部化」后交给 Node 原生加载，Node 不认这些写法就炸了。

### 修法：把该编译的依赖加进 noExternal

```js
// vite.config.js
export default defineConfig({
  ssr: {
    // 强制这些依赖被 Vite 编译、打进 SSR 包，而不是丢给 Node 直接 require
    noExternal: ['some-esm-only-lib', '@company/ui'],
  },
});
```

反方向也有：某依赖被 Vite 错误地打进包、反而出问题（比如它依赖 Node 原生扩展），用 `ssr.external` 强制让它保持外部化。

> 排查口诀：**SSR 报** `ERR_REQUIRE_ESM` **/** `Unexpected token 'export'` **→ 找出报错的那个包，丢进** `ssr.noExternal`**。** monorepo 的内部包（见 [02 节](./02-monorepo与workspace的Vite组织与依赖处理.md)）几乎总是要 `noExternal`。

## 四、坑三：条件导入（只在某一端跑的代码）

### 现象

- 服务端：`window is not defined` / `document is not defined`（代码在 Node 里碰了浏览器 API）。
- 客户端：`process is not defined`（碰了 Node API）。

### 根因

某段代码只在一个世界有意义（操作 DOM 只在浏览器、读 `process.env` 只在 Node），却在两个世界都被执行了。

### 修法：用 import.meta.env.SSR 隔离，让无关分支被裁掉

Vite 提供 `import.meta.env.SSR` 这个布尔常量：**在 SSR 构建里是** `true`**，在客户端构建里是** `false`**，且它是「编译期常量」——被** `if (!import.meta.env.SSR)` **判掉的分支会在对应构建里被当死代码删除。**

```js
export function platformLabel() {
  if (import.meta.env.SSR) {
    // 这段只进 SSR 包；客户端构建里整段被裁掉，
    // 所以这里用 Node 专有的 process 也不会让客户端报错。
    return `服务端（Node ${process.version})`;
  }
  // 这段只在客户端有效
  return navigator.userAgent;
}
```

对「只在某一端才有的**模块**」，用动态 import 做条件加载，避免在错误的世界 import 进来：

```js
// 只在服务端加载某个 Node-only 模块
const fsMod = import.meta.env.SSR ? await import('node:fs/promises') : null;
```

> 对比 `typeof window !== 'undefined'`：它是**运行期**判断，两个分支的代码都会被打进包（只是运行时挑一条走）。`import.meta.env.SSR` 是**编译期**判断，无关分支直接被删——更干净、能让 Node-only / browser-only 代码真正不出现在错误的产物里。优先用后者。

## 五、坑四：环境变量泄露（最危险的一个）

### 现象

最隐蔽也最严重：你的数据库密码、API 密钥，出现在了浏览器「查看源代码」就能看到的 HTML 或 JS 里。

### 根因

SSR 同时有「服务端环境变量」和「会被打进客户端的变量」两个概念，混淆了就会泄露。Vite 的规则非常明确：

- **只有** `VITE_` **前缀的变量**会被注入 `import.meta.env`、**打进客户端包**（公开可见）。
- **不带前缀的变量**不进 `import.meta.env`，只能在**服务端**通过 `process.env` 读到。

```bash
# .env
VITE_PUBLIC_TITLE=我的应用      # 会进客户端，浏览器可见 → 放公开信息
SECRET_API_KEY=super-secret     # 只服务端 process.env 可读 → 放密钥
```

泄露通常发生在两种操作：

1. 给密钥误加了 `VITE_` 前缀 → 它被打进客户端包。
2. 服务端把含密钥的对象**整个序列化**进了 `window.__SSR_STATE__` → 密钥跟着进了 HTML。

### 修法

```js
// ✅ 服务端：用密钥去取数据，但只把「结果/脱敏后的数据」放进下发的 state
export async function getServerState() {
  const key = process.env.SECRET_API_KEY;   // 服务端读，不带 VITE_ 前缀
  const data = await fetchWithKey(key);
  return {
    data,                  // 可以下发
    // ❌ 绝不要写 apiKey: key —— 它会被序列化进 HTML
  };
}
```

> 铁律：**密钥永远不带** `VITE_` **前缀，永远不进** `import.meta.env`**，永远不进下发给客户端的** `state`**。** 客户端要用的「公开配置」才加 `VITE_`。serialize state 前先问自己一句：这个对象里有没有夹带敏感字段？

补充：服务端进程的 `process.env` 由谁填？真实生产里是 dotenv / 进程管理器（pm2、systemd）/ 容器环境变量注入；Vite 的 `loadEnv(mode, root, '')`（第三参数空字符串=不限前缀全部加载）可在你的 SSR server 启动脚本里把 `.env` 里的非前缀变量也读进来用。

## 六、动手：四大坑位的可观察现场

配套 demo 把其中三个坑（水合不一致、条件导入、环境变量泄露）做成「能在浏览器里亲眼看到」的现场：红框=水合不一致、绿框=一致，控制台打印各项检测结果；外部化坑以 `vite.config.js` 注释 + 本节讲解呈现（它通常要在 production SSR 构建里才触发）。

配套代码：[第一阶段-使用篇/04-多环境与工程化场景/03-SSR工程坑位/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/04-多环境与工程化场景/03-SSR工程坑位/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/04-多环境与工程化场景/03-SSR工程坑位
npm install
npm run dev     # http://localhost:5173，务必打开浏览器控制台
```

观察点：

1. **水合不一致**：`#buggy` 块被画红框、控制台 `[hydration mismatch]`（服务端/客户端各算各的 `new Date()`）；`#stable` 块绿框、`[hydration ok]`（时间经 `window.__SSR_STATE__` 下发后客户端复用）。
2. **条件导入**：`platform.js` 里用了 Node 专有的 `process`，`entry-client.js` 也 `import` 并调用了它，但客户端控制台打印的是「客户端水合（navigator…）」、**不报** `process is not defined`——因为那段被 `import.meta.env.SSR` 包住、客户端构建时已被 DCE 裁掉（`#platform` 块服务端显示「服务端渲染（Node …）」属预期差异）。
3. **环境变量泄露**：控制台里 `VITE_PUBLIC_TITLE` 有值、`SECRET_API_KEY` 是 `undefined`——只有带前缀的进了客户端；服务端能读到密钥（页面显示「读到密钥=true」）但只下发布尔值。
4. **外部化**：看 `vite.config.js` 里 `ssr.noExternal` / `ssr.external` 的注释，对照本节坑二。

## 七、本节小结

- SSR 一切坑的根源：同一份代码在 Node 和浏览器**各执行一次**。
- 水合不一致：服务端/客户端渲染结果不同（时间、随机数、`typeof window` 分支）。修法是服务端算好、序列化进 HTML（`window.__SSR_STATE__`）、客户端复用。
- 服务端外部化：`ERR_REQUIRE_ESM` / `Unexpected token 'export'` 多半是某依赖该编译却被外部化了，加进 `ssr.noExternal`；monorepo 内部包几乎总要 `noExternal`。
- 条件导入：用编译期常量 `import.meta.env.SSR` 隔离只在某一端跑的代码（无关分支被裁掉），优于运行期的 `typeof window` 判断。
- 环境变量泄露（最危险）：只有 `VITE_` 前缀变量进客户端；密钥不加前缀、不进 `import.meta.env`、不进下发的 state。

## 八、可直接用于项目的 checklist

- [ ] 水合报错先查：是否用了 `new Date()`/`Math.random()`/`typeof window` 这类「两次执行必不同」的值；改为服务端下发 state、客户端复用。
- [ ] SSR 报 `ERR_REQUIRE_ESM`/`Unexpected token 'export'`：定位报错依赖，加进 `ssr.noExternal`。
- [ ] monorepo 内部 workspace 包在 SSR 下默认加进 `ssr.noExternal`。
- [ ] 只在某一端跑的代码用 `import.meta.env.SSR` 包裹（而非 `typeof window`），Node-only / browser-only 代码不出现在错误产物里。
- [ ] 密钥类变量**绝不**加 `VITE_` 前缀，服务端用 `process.env` 读。
- [ ] 序列化下发 state 前，逐字段确认没有夹带密钥/敏感信息。
- [ ] SSR server 启动脚本用 `loadEnv(mode, root, '')` 或进程环境注入非前缀变量。