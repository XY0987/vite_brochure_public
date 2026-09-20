# 06 · SSR、库模式、多页应用等进阶场景

> 本节解决的真实工程问题：同一个 Vite，怎么从“做单页应用”切换到“做服务端渲染”“发一个 npm 库”“做多入口站点”？这三件事各自要改什么配置、产物长什么样、有哪些专属坑？本节用三个最小可运行 demo 把它们讲透，让你需要时能直接照搬。

事实基线：2026 年中，Vite 8.x。SSR 多环境的进一步演进（Environment API 下的 client/ssr/edge 多运行时）属于第一阶段后续「多环境与工程化」章节与第二/三阶段，本节聚焦最常用的单 SSR 场景。

---

## 一、SSR：服务端渲染

### 它解决什么

SSR 让首屏 HTML 在服务端生成后直接返回，利于 SEO 和首屏速度。Vite 的角色是：**在开发期用中间件模式（middleware mode）把按需编译能力嵌进你的 Node 服务器**，在生产期分别构建 client 和 server 两份产物。

本节的 SSR demo 不是让你在日常业务里从零手写 SSR 框架，而是用一个最小 Node 服务把机制拆开：浏览器请求进来后，Node 服务负责接请求、拼 HTML、返回响应；Vite 在开发期负责处理 HTML、按需编译服务端入口、提供 HMR。真实项目里，这些流程通常由 Nuxt / SvelteKit / Astro 等框架封装好，你需要理解的是它们底层大致在做什么。

可以把分工先记成一句话：**Vite 提供 SSR 需要的编译、构建和开发体验能力；真正的 HTTP 服务、路由处理、数据获取、HTML 拼装与响应返回，要由框架或你自己的 Node 服务完成**。所以 SSR 不是“打开 Vite 某个开关就自动完成”，而是 Vite 给这套服务端渲染流程提供工具链支撑。

### 关键概念

- **两套构建**：SSR 生产构建通常要跑两次命令：`vite build` 生成 client 产物（给浏览器 hydrate），`vite build --ssr src/entry-server.js` 生成 server 产物（给 Node 跑渲染逻辑）。
- **`server.middlewareMode: true`**：只用于**开发期 SSR**。场景是你自己起一个 Express/Koa 等 Node 服务，Vite 不单独监听端口，而是作为中间件挂到这个服务上；这样同一个服务既能处理 SSR 请求，又能使用 Vite 的 HMR、模块转换能力。
- **`vite.ssrLoadModule(url)`**：这是**开发期 SSR** 的常见入门 API，用来在 Node 里按需加载并执行服务端入口模块（带 HMR、按需编译）。生产运行时通常直接加载 `vite build --ssr` 生成的 server bundle。Vite 8 源码里已经给它接上了基于 Module Runner 的兼容实现，并把它列入 future deprecation；新框架/底层运行时会逐步转向 Environment Runner / Module Runner，但本节作为最小 SSR 入门仍保留这个写法，并提醒你不要把它当成长期底层扩展点。
- **`import.meta.env.SSR`**：代码里判断当前是否运行在服务端，用于隔离“只能在浏览器/只能在 Node 跑”的逻辑。

### 典型 dev 流程（伪代码）

```js
const vite = await createViteServer({
  server: { middlewareMode: true },
  appType: 'custom', // 不接管 index.html，由你控制 HTML 拼装
});
app.use(vite.middlewares);

app.use('*', async (req, res) => {
  // 1. 读 index.html 模板，交给 vite 注入 HMR client 等
  let template = await vite.transformIndexHtml(req.originalUrl, rawTemplate);
  // 2. 按需加载服务端入口（带编译/HMR）
  const { render } = await vite.ssrLoadModule('/src/entry-server.js');
  // 3. 执行渲染，把结果塞进模板
  const appHtml = await render(req.originalUrl);
  res.end(template.replace('<!--ssr-outlet-->', appHtml));
});
```

### SSR 专属坑（先知道，详见 [04-03 SSR 工程坑位排查手册](../04-多环境与工程化场景/03-SSR工程坑位排查手册.md)）

- **客户端水合不一致**：服务端和客户端渲染出的 DOM 不一致会报 hydration mismatch。常见于用了 `Date.now()`、随机数、`window` 判断分支。
- **服务端外部化（externalize）**：Node 端不需要打包 `node_modules` 依赖，Vite 默认外部化它们；但某些包需要 `ssr.noExternal` 强制打包。
- **环境变量泄露**：服务端的密钥别通过 SSR 渲染进 HTML（呼应 [03 环境变量](./03-devserver构建预览与环境变量多模式.md)）。

这里的 externalize 可以理解成“server bundle 不把某个依赖打进去，运行时交给 Node 去 `node_modules` 里加载”。如果某个包需要被 Vite 转换，或外置后 ESM/CJS、条件导出解析不对，就用 `ssr.noExternal` 告诉 Vite：这个包不要外置，强制打进 server bundle。环境变量泄露则是另一类问题：SSR 代码能读服务端密钥，但 SSR 返回的 HTML 最终会发给浏览器，密钥一旦被拼进 HTML 或初始状态对象里，就等于公开了。

> 实战中多数人不会从零搭 SSR，而是用 Nuxt / SvelteKit / Astro 这类基于 Vite 的框架。但理解上面的机制，能让你在框架出问题时知道底层发生了什么。

## 二、库模式（Library Mode）：发一个 npm 包

### 它解决什么

你要把一段代码发布成 npm 库给别人 `import`，而不是做一个网站。库模式让 Vite 产出**多种格式（ESM / UMD / CJS）、不打包 peer 依赖、生成正确入口**的产物。

### 核心配置 `build.lib`

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { fileURLToPath } from 'node:url';

// ESM 配置里用 import.meta.url 取路径，别用 __dirname（见 09 章）。
const r = (p: string) => fileURLToPath(new URL(p, import.meta.url));

export default defineConfig({
  build: {
    lib: {
      entry: r('./src/index.js'),
      name: 'MyLib',                 // UMD 全局变量名
      fileName: (format) => `my-lib.${format}.js`,
      formats: ['es', 'umd'],        // 产出 ESM + UMD
    },
    rolldownOptions: {
      // 关键：把 peer 依赖外部化，不打进库里（否则用户项目里会有两份 React 等）
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' },     // UMD 下外部依赖的全局名
      },
    },
  },
});
```

### 配套 `package.json` 入口字段（同样关键）

库能不能被正确 import，一半靠产物、一半靠 `package.json` 的入口声明：

```json
{
  "name": "my-lib",
  "type": "module",
  "files": ["dist"],
  "main": "./dist/my-lib.umd.js",
  "module": "./dist/my-lib.es.js",
  "exports": {
    ".": {
      "import": "./dist/my-lib.es.js",
      "require": "./dist/my-lib.umd.js"
    }
  }
}
```

### 库模式专属点

- **CSS**：库里若有样式，默认会单独产出一个 CSS 文件，需要让用户手动 import（或配 `cssCodeSplit`/插件注入）。
- **类型声明**：Vite 不生成 `.d.ts`，要用 `tsc --declaration --emitDeclarationOnly`（Vue 项目用 `vue-tsc --declaration --emitDeclarationOnly`）或 `vite-plugin-dts` 单独产出（呼应 [07 TypeScript](./07-TypeScript处理.md) 的“Vite 只转译”事实）。
- **external**：忘了把框架/大依赖 external 是最常见的库膨胀根因。

## 三、多页应用（MPA）：多个 HTML 入口

### 它解决什么

不是所有项目都是 SPA。后台系统常有 `index.html`、`admin.html`、`login.html` 多个独立页面，各自有入口脚本。Vite 8 推荐通过 `build.rolldownOptions.input` 配多入口支持（旧名 `rollupOptions.input` 仍有兼容层）。

### 核心配置

```ts
import { defineConfig } from 'vite';
import { fileURLToPath } from 'node:url';

const r = (p: string) => fileURLToPath(new URL(p, import.meta.url));

export default defineConfig({
  build: {
    rolldownOptions: {
      input: {
        main: r('./index.html'),
        admin: r('./admin.html'),
        login: r('./login/index.html'),
      },
    },
  },
});
```

### MPA 要点

- **每个 HTML 是独立入口**，各自的依赖图分别打包，公共依赖会被自动提取为共享 chunk。
- dev 期直接访问对应路径即可：`/admin.html`、`/login/`。
- 目录结构决定产物路径：`login/index.html` 会产出到 `dist/login/index.html`。
- 入口 HTML 必须在项目内可被解析到（注意 `root` 设置）。

## 四、动手：三个子项目

配套代码：[第一阶段-使用篇/01-核心使用/06-SSR库模式多页应用/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/01-核心使用/06-SSR库模式多页应用/)，内含三个独立子目录：

```
06-SSR库模式多页应用/
├── ssr/   # 基于 Express + vite middleware 的最小 SSR
├── lib/   # 库模式：产出 ESM + UMD
└── mpa/   # 多页应用：三个 HTML 入口
```

### SSR

```bash
cd ssr
npm install
npm run dev      # 启动带 Vite 中间件的 Node 服务器，访问 http://localhost:5173
# 查看页面源代码：首屏内容是服务端渲染的，不是空 div
```

### 库模式

```bash
cd lib
npm install
npm run build    # 产出 dist/my-lib.es.js 与 my-lib.umd.js
ls dist          # 观察多格式产物
```

### 多页应用

```bash
cd mpa
npm install
npm run dev      # 访问 / 、/admin.html 、/login/
npm run build    # 观察 dist 下分别产出三个页面与共享 chunk
```

## 五、本节小结

- **SSR**：dev 用 `middlewareMode` + `ssrLoadModule` 把 Vite 嵌入你的 Node 服务器；生产分别构建 client 与 server。注意水合一致性、外部化、变量泄露三类坑。
- **库模式**：`build.lib` 定义入口与格式，`rolldownOptions.external` 外部化 peer 依赖，`package.json` 的 `exports`/`main`/`module` 决定能否被正确 import；类型声明需另外生成。
- **多页应用**：`rolldownOptions.input` 配多 HTML 入口，公共依赖自动提取，产物路径跟随目录结构。

## 六、可直接用于项目的 checklist

SSR：
- [ ] dev 用 `server.middlewareMode` + `appType: 'custom'`，HTML 经 `transformIndexHtml` 处理。
- [ ] 区分 client / server 两次构建；用 `import.meta.env.SSR` 隔离平台相关代码。
- [ ] 检查过水合一致性（无随机/时间/`window` 导致的首屏差异）与密钥不泄露。

库模式：
- [ ] `build.lib` 配了 `entry`/`formats`；peer 依赖在 `rolldownOptions.external` 里。
- [ ] `package.json` 配了 `exports`/`main`/`module`/`files`。
- [ ] 用 `vite-plugin-dts` 或 `tsc` 单独生成了 `.d.ts`。

多页应用：
- [ ] 每个页面在 `rolldownOptions.input` 里登记；产物路径与目录结构一致。
- [ ] 确认 `root`/`base` 设置使所有 HTML 入口都能被解析与正确部署。
