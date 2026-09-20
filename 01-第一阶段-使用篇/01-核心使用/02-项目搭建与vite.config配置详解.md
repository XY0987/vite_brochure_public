# 02 · 项目搭建、目录约定与 `vite.config` 完整配置详解

> 本节解决的真实工程问题：新建一个 Vite 项目五分钟搞定，但半年后 `vite.config.ts` 里塞了二十几项配置，没人讲得清每一项为什么在、删了会怎样。本节给你一份**逐项注释、能讲清取舍**的配置范本，以及 Vite 的目录约定心智模型——让配置文件从“能跑就别动”变成“知道每项在管什么”。

事实基线：2026 年中，Vite 8.x。

---

## 一、项目搭建：脚手架与手动两条路

### 1. 用官方脚手架（推荐起步）

```bash
npm create vite@latest my-app
# 按提示选择框架（vanilla / vue / react / svelte / solid / qwik...）与变体（TS / JS）
cd my-app
npm install
npm run dev
```

`npm create vite` 背后是 `create-vite` 脚手架，它只做一件事：**拷贝一套模板**。它不是魔法，生成的就是普通文件，你完全可以读懂、改写。

### 2. 手动初始化（理解每个文件从哪来）

脚手架隐藏了细节。手动建一遍，你才知道一个最小 Vite 项目到底需要什么：

```bash
mkdir my-app && cd my-app
npm init -y
npm install -D vite
```

然后只需要两个文件就能跑起来：

```html
<!-- index.html （注意：在项目根目录，不是 public/） -->
<!doctype html>
<html>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

```js
// src/main.js
document.querySelector('#app').textContent = 'Hello Vite';
```

`npx vite` 就能启动。**注意：连 `vite.config` 都不是必需的**——没有配置文件时 Vite 用一套合理默认值工作。配置文件是“需要时才加”的东西，不是仪式。

## 二、目录约定：Vite 的几个“知道了就不踩坑”的规则

Vite 约定优于配置，但有几条约定不知道就会反复踩：

### 1. `index.html` 是入口，且在项目根目录

这是 Vite 最反直觉的一点。在 webpack 里 `index.html` 通常是 `html-webpack-plugin` 的产物模板；在 **Vite 里 `index.html` 是真正的源码入口**，位于项目根目录（不是 `public/`、不是 `src/`）。Vite 把它当作模块图的起点，解析里面的 `<script type="module">` 和 `<link>`。

> 多页应用时会有多个 html 入口，见 [06 多页应用](./06-SSR库模式多页应用.md)。

### 2. `public/` 目录：原样拷贝、不经处理

放进 `public/` 的文件**不会被 Vite 处理或加哈希**，构建时原样拷到产物根目录。

- 引用方式：用根绝对路径 `/favicon.ico`，**不要 import**。
- 适用：`robots.txt`、`favicon`、第三方不希望被处理的脚本。
- 不适用：能被代码 import 的图片/字体——那些应放 `src/` 走资源处理流程拿哈希（见 [04 静态资源](./04-静态资源CSS别名与路径解析.md)）。

### 3. `src/`：你的源码，走完整编译/HMR 流程

约定俗成放源码。它不是硬性规定，但脚手架与社区都默认如此。

### 4. 产物默认输出到 `dist/`，如需修改可配置 `build.outDir`

## 三、`vite.config` 配置文件详解

配置文件支持 `vite.config.js` / `.ts` / `.mjs` 等。**强烈推荐用 `.ts` + `defineConfig`**，可获得完整类型提示——这是“讲清每一项”的前提。

下面按**功能分组**讲解最常用的配置项。完整可运行版本见配套代码 `02-项目搭建与配置详解/vite.config.ts`。这份配置不是官方配置全集，而是日常项目最常用的一组配置范本；更多高级配置会在后续章节按场景展开。

### 1. `defineConfig`：拿到类型提示

```ts
import { defineConfig } from 'vite';

export default defineConfig({
  // ... 这里每个 key 都有类型提示和文档
});
```

`defineConfig` 本身不做任何运行时逻辑，纯粹是个**类型包裹**。它还支持传入函数，根据命令/模式返回不同配置（见 [03 多模式](./03-devserver构建预览与环境变量多模式.md)）。

### 2. 根与路径相关

```ts
{
  root: process.cwd(),   // 项目根（index.html 所在处）。默认当前目录，单仓多项目时常改
  base: '/',             // 部署的公共基础路径。子路径部署（如 /admin/）时必须改，否则资源 404
  publicDir: 'public',   // 静态原样拷贝目录，设为 false 可禁用
}
```

`base` 是最常见的“线上资源 404”根因：项目部署在 `https://x.com/app/` 下，却没把 `base` 设成 `/app/`，于是产物里引用的 `/assets/xxx.js` 全部指向了域名根。

### 3. 路径别名 `resolve.alias`

```ts
import { fileURLToPath, URL } from 'node:url';

{
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
}
```

让 `import Foo from '@/components/Foo'` 代替一长串 `../../../`。注意：**别名只解决打包/编译时的解析，TS 的类型解析要在 `tsconfig.json` 的 `paths` 里同步配一份**，两边都要有，否则要么编译报错、要么类型报错。详见 [04 别名与路径解析](./04-静态资源CSS别名与路径解析.md)。

### 4. 插件 `plugins`

```ts
import vue from '@vitejs/plugin-vue';

{
  plugins: [vue()],
}
```

框架支持（Vue/React/Svelte）、各种能力增强都通过插件接入。**插件有顺序与执行边界**（`enforce: 'pre' | 'post'`、`apply: 'serve' | 'build'`），这是第三大章「插件与生态」的主题，这里先知道“框架支持靠插件”即可。

### 5. dev server `server`

```ts
{
  server: {
    port: 5173,
    open: true,          // 启动自动开浏览器
    host: true,          // 监听 0.0.0.0，方便手机/局域网访问
    proxy: {             // 开发环境反向代理，解决跨域
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
        rewrite: (p) => p.replace(/^\/api/, ''),
      },
    },
  },
}
```

`proxy` 是前后端分离开发的高频配置：前端请求 `/api/xxx`，dev server 转发到后端，浏览器侧无跨域。**注意 `proxy` 只在 dev/preview 生效，生产环境的跨域要靠真正的网关/反代解决**——这是新手常见误解。

### 6. 构建 `build`

```ts
{
  build: {
    outDir: 'dist',
    sourcemap: false,        // 线上排错需要时设 true 或 'hidden'（见 09 节）
    target: 'baseline-widely-available', // 目标浏览器基线，Vite 8 的现代默认
    // Vite 8 底层使用 Rolldown：分包推荐用 rolldownOptions.output.codeSplitting
    rolldownOptions: {       // 兼容层仍接受旧名 rollupOptions（自动映射到 rolldown）
      output: {
        // manualChunks 在 Vite 8 推荐迁移为 codeSplitting，见 08 节
      },
    },
  },
}
```

> 版本提示：Vite 7 可以通过 `rolldown-vite` 试用 Rolldown，Vite 8 默认使用 Rolldown。老项目里的 `build.rollupOptions` 大多还能继续工作；但如果要配置分包，建议优先使用 `rolldownOptions.output.codeSplitting`，早期过渡字段 `advancedChunks` 已不再推荐。迁移要点见 [08 性能优化](./08-性能优化产物分析与报错排查手册.md)。

### 7. 环境变量与 `envPrefix`

```ts
{
  envPrefix: 'VITE_',   // 只有以此前缀开头的变量才会暴露到客户端代码（默认 VITE_）
}
```

这是**防止密钥泄露的安全闸门**：没有前缀的环境变量不会被注入到浏览器代码里。详见 [03 环境变量](./03-devserver构建预览与环境变量多模式.md)。

### 8. CSS 相关 `css`

预处理器、CSS Modules、PostCSS 都在这里，单列一节讲，见 [04](./04-静态资源CSS别名与路径解析.md)。

### 9. 依赖预构建 `optimizeDeps`

控制第三方依赖的预打包行为，单列一节讲，见 [05](./05-依赖预构建.md)。

## 四、配置怎么被加载（简单理解）

不深入源码（那是第二阶段），但用层面需要知道：

1. Vite 启动时找 `vite.config.*`，**自己先用内置工具把这个 TS/ESM 配置文件转译并执行**（所以配置文件里能用 `import`、`__dirname` 的 ESM 等价物）。
2. 把你的配置和**内置默认值、各插件提供的配置**做深合并。
3. 区分 `command`（`serve`/`build`）与 `mode`（`development`/`production` 等），可用函数形式按条件返回不同配置。

> 这解释了一个常见疑惑：为什么 `vite.config.ts` 里不能直接用 CommonJS 的 `__dirname`——因为它被当作 ESM 处理，要用 `fileURLToPath(import.meta.url)`。

## 五、动手：一份逐项注释的配置

配套代码：[第一阶段-使用篇/01-核心使用/02-项目搭建与配置详解/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/01-核心使用/02-项目搭建与配置详解/)

这是一个能跑的最小项目，`vite.config.ts` 里每一项都有注释说明“管什么、默认值、什么时候改”。

```bash
cd vite_brochure_code_public/第一阶段-使用篇/01-核心使用/02-项目搭建与配置详解
npm install
npm run dev      # 体验 server.port / open / alias
npm run build    # 观察 dist 产物与 base 的影响
npm run preview
```

建议动手实验：
- 把 `base` 改成 `/sub/` 再 `build` + `preview`，观察资源路径变化（以及不配会怎样 404）。
- 把 `@` 别名删掉，看 `import '@/ui.js'` 如何报错——理解别名解决的是什么。

## 六、本节小结

- 脚手架只是“拷模板”，最小 Vite 项目只要 `index.html` + 一个入口模块，配置文件按需才加。
- 记住目录约定：`index.html` 在根目录且是真入口；`public/` 原样拷贝用绝对路径引用；产物默认 `dist/`。
- 用 `defineConfig` + `.ts` 拿类型提示；配置按 root/path、plugins、server、build、env、css、optimizeDeps 分组理解。
- `base`（子路径部署）、`resolve.alias`（同时配 tsconfig）、`server.proxy`（仅 dev）是三个最容易出错的点。

## 七、可直接用于项目的配置 checklist

新建或接手一个 Vite 项目时逐条过：

- [ ] 配置文件用 `.ts` + `defineConfig`，享受类型提示。
- [ ] 确认部署路径：非根路径部署时 `base` 已正确设置。
- [ ] 别名在 `vite.config` 的 `resolve.alias` 与 `tsconfig.json` 的 `paths` **两处都配**了。
- [ ] 需要后端联调时配了 `server.proxy`，并清楚它**生产不生效**。
- [ ] 敏感变量没有用 `VITE_` 前缀（不会泄露到客户端）。
- [ ] `public/` 里只放无需处理的文件，且用 `/xxx` 绝对路径引用、不 import。
- [ ] 升级 Vite 8 时确认分包配置（`manualChunks` → `codeSplitting`，`advancedChunks` 仅为已废弃过渡名）的迁移计划。
