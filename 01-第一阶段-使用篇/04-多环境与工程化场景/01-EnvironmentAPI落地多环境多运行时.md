# 01 · Environment API 落地：从单一 client/ssr 到 client / ssr / edge 多环境多运行时

> 本节解决的真实工程问题：以前 Vite 的世界很简单——要么打个浏览器包（client），要么再加个服务端包（ssr），两条线就够了。但今天一个稍微正经点的应用，产物可能要同时跑在**浏览器、Node 服务端、还有边缘运行时（Cloudflare Workers / Vercel Edge / Deno）**——三个目标、三套全局 API、三种依赖解析规则。在 Vite 6 之前，这种「多目标」只能靠跑多次 `vite build`、维护多份 config、用一堆 `if (ssr)` 手动打补丁，既乱又容易出「dev 和某个环境行为不一致」的诡异 bug。Vite 6 引入的 **Environment API** 就是为收拾这个局面而生：把「构建目标」正式抽象成一等公民「环境（Environment）」，每个环境有独立的模块图、解析规则和产物。本节带你把它真正落地——用**一份配置**并发产出 client / ssr / edge 三套产物，且同一份业务源码在三个环境里跑各自的运行时实现。

事实基线：2026 年中，Vite 8.x（默认 Rolldown 打包器、Oxc 转译）。Environment API 于 Vite 6 引入，Vite 8 已是稳定的一等能力。

> 范围说明：本节讲**怎么用**（第一阶段定位）。Environment API 的内部实现（`DevEnvironment`、每环境独立模块图怎么重构）见第二阶段，它「为什么这么设计」见第三阶段。这里只回答：它长什么样、我该怎么配、能解决我什么问题。

---

## 一、先理解「环境」是什么：一次心智升级

在 Environment API 之前，你脑子里的 Vite 大概是这样：

```
源码 ──┬──> client 包（给浏览器）
       └──> ssr 包（给 Node，可选）
```

`client` 和 `ssr` 是**写死的两条特殊路径**。你想加个「给边缘运行时的包」？对不起，没有官方位置，只能自己折腾。

Environment API 把这件事彻底重构成：

```
源码 ──> [client 环境] ──> client 产物
     ──> [ssr 环境]    ──> ssr 产物
     ──> [edge 环境]   ──> edge 产物   ← 你可以自定义任意多个环境
```

**「环境」= 一个独立的构建/运行目标，拥有自己的：模块图、`resolve` 规则（条件、别名）、`build` 输出配置、甚至自己的插件管线行为。** `client` 和 `ssr` 只是 Vite 内置的两个默认环境，不再是特殊待遇——你可以照着它们的样子，自定义 `edge`、`rsc`、`workerd` 等任意环境。

> 一句话抓住本质：**以前是「一份配置 + 两条硬编码分支」，现在是「一份配置 + N 个对等环境」。** 这就是从「单一 client/ssr」到「多环境」的升级。

## 二、最小多环境配置：声明三个环境

落地入口是 `vite.config` 里的两个新字段：`environments`（声明有哪些环境）和 `builder.buildApp`（控制 build 时怎么编排这些环境）。

```js
// vite.config.js
import { defineConfig } from 'vite';

export default defineConfig({
  environments: {
    // 浏览器环境：内置默认环境，产出现代 ESM 包。
    client: {
      // 多环境构建下，client 也要显式给出 HTML 入口（否则 builder 不知道从哪进）。
      build: {
        outDir: 'dist/client',
        rolldownOptions: { input: './index.html' },
      },
    },
    // Node SSR 环境：内置默认环境。
    ssr: {
      build: {
        outDir: 'dist/ssr',
        ssr: true,
        rolldownOptions: { input: './src/entry-server.js' },
      },
    },
    // 边缘环境：完全自定义的新环境。
    edge: {
      resolve: {
        // 关键：边缘运行时按 worker/edge 条件解析第三方包的 exports。
        conditions: ['worker', 'edge'],
        noExternal: true,   // 边缘产物要自包含，不外部化依赖
      },
      build: {
        outDir: 'dist/edge',
        ssr: true,
        rolldownOptions: { input: './src/entry-edge.js' },
      },
    },
  },
});
```

每个环境就是 `environments` 下的一个键，值里你能配 `build`（输出）、`resolve`（解析规则）等。**`client` / `ssr` 是预置名字，写上去就是覆盖它们的默认配置；`edge` 是你凭空加的——这正是「多环境」可扩展的地方。**

这里的 `resolve.conditions: ['worker', 'edge']` 不是在代码里注入什么变量，而是影响**依赖包入口怎么选**。现在很多包会在 `package.json` 的 `exports` 里给不同运行时准备不同入口：

```json
{
  "exports": {
    ".": {
      "worker": "./dist/worker.js",
      "edge": "./dist/edge.js",
      "node": "./dist/node.js",
      "default": "./dist/index.js"
    }
  }
}
```

当边缘环境配置了 `conditions: ['worker', 'edge']`，Vite 解析第三方包时会优先匹配 `worker` / `edge` 入口，而不是误走可能依赖 `fs`、`net`、`process` 等 Node 能力的 `node` 入口。简单说，它是在告诉构建器：**这份产物要跑在 Worker / Edge Runtime，请按这个运行时选择依赖实现。**

## 三、多运行时：同一份源码，按环境解析到不同实现

「多环境」只是把产物拆开；**「多运行时」才是关键——同一份业务代码，在不同环境里要调用不同的平台 API**（浏览器有 `document`、Node 有 `process`、边缘只有 Web 标准 API）。怎么做到「一份源码、三套实现」而不写满 `if`？

思路：业务代码只 import 一个**抽象的平台入口**，由各环境决定它解析到哪个具体实现。

```js
// src/shared/app.js —— 三个环境共享的同一份业务代码
import { runtime, now } from '#platform';   // 抽象入口，不关心自己跑在哪

export function renderHTML(url) {
  return `<h1>Hello from ${runtime}</h1><p>${now()}</p>`;
}
```

`#platform` 在三个环境里要分别落到 `browser.js` / `node.js` / `edge.js`。落地这件事，最贴近真实多环境插件的写法是用一个插件，在 `resolveId` 钩子里读 **`this.environment`**（Environment API 给插件上下文新增的「当前正在构建哪个环境」信息）来分流：

```js
// vite.config.js 里的一个小插件
function platformResolver() {
  const map = {
    client: r('./src/platform/browser.js'),
    ssr: r('./src/platform/node.js'),
    edge: r('./src/platform/edge.js'),
  };
  return {
    name: 'platform-resolver',
    resolveId(id) {
      if (id === '#platform') {
        // this.environment.name = 'client' | 'ssr' | 'edge'
        const envName = this.environment?.name ?? 'client';
        return map[envName] ?? map.client;
      }
    },
  };
}
```

> 记住这个 `this.environment`：它是 Environment API 在「插件视角」最重要的入口。一个插件能感知自己**正在为哪个环境工作**，从而对不同环境做不同的事——这是过去那套「全局 `ssr` 布尔」做不到的精细控制。

这里顺手补一个实践判断：**插件运行在构建阶段，通常还是 Node 进程；风险不在“插件会不会跑到 Edge 里”，而在“插件会不会给 Edge 产物注入 Node 专属代码，或把依赖解析到 Node 入口”。**

所以多环境插件配置可以按差异大小分两种：

- **插件本身环境无关，只是不同环境下行为略有不同**：用一套插件配置，在插件内部通过 `this.environment.name` 分流。比如同一个 `platformResolver` 根据 `client` / `ssr` / `edge` 返回不同平台实现。
- **不同平台依赖的插件完全不同，或插件有明显运行时副作用**：按环境拆配置。比如 A 平台需要 Worker 适配插件，B 平台需要 Node SSR 插件；或者某个插件会自动注入 `node:fs`、改写 `external`、补 polyfill，就不要强行共享。

我的经验判断是：**行为差异小，用 Environment API 分流；依赖差异大、产物约束不同、插件副作用不可控，就配置分离。**

三份平台实现各自用各自运行时能用的 API：

```js
// src/platform/node.js —— 只会被打进 ssr 包，可放心用 Node API
export const runtime = `node ${process.version}`;
export const now = () => new Date().toISOString();

// src/platform/edge.js —— 打进 edge 包，只用 Web 标准 API
export const runtime = 'edge (Web/Worker runtime)';
export const now = () => new Date().toISOString();
```

这样，`app.js` 一字不改，在 ssr 环境里 `runtime` 是 Node 版本号、在 edge 环境里是边缘标识——**一份源码、多运行时**。

## 四、并发构建：builder.buildApp 编排多个环境

声明了多个环境后，`vite build` 怎么知道要把它们都构建出来、按什么顺序？答案是 `builder.buildApp`：

```js
export default defineConfig({
  // ...environments...
  builder: {
    async buildApp(builder) {
      // builder.environments 拿到所有声明过的环境
      await Promise.all([
        builder.build(builder.environments.client),
        builder.build(builder.environments.ssr),
        builder.build(builder.environments.edge),
      ]);
    },
  },
});
```

`buildApp` 是「构建编排器」：你在这里决定**先构建谁、后构建谁、能不能并发**。上面用 `Promise.all` 让三个环境**并发构建**；若环境间有依赖（比如 ssr 产物依赖 client 的 manifest），就改成顺序 `await`。不写 `buildApp` 时，Vite 默认按声明顺序逐个构建。

跑一次 `vite build`，终端会看到三条环境并行推进：

```
vite v8.1.0 building client environment for production...
vite v8.1.0 building ssr environment for production...
vite v8.1.0 building edge environment for production...
✓ built in 18ms   (dist/client)
✓ built in 18ms   (dist/ssr)
✓ built in 18ms   (dist/edge)
```

三套产物分别落到 `dist/client`、`dist/ssr`、`dist/edge`。**一条命令、一份配置，多环境多运行时产物一次到位**——这就是 Environment API 兑现的核心价值。

## 五、动手：跑通三环境三运行时

配套 demo 把上面所有片段集成成一个可跑项目，你能亲眼看到「同一份 `app.js`」在三个运行时下的不同输出。

配套代码：[第一阶段-使用篇/04-多环境与工程化场景/01-EnvironmentAPI多环境/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/04-多环境与工程化场景/01-EnvironmentAPI多环境/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/04-多环境与工程化场景/01-EnvironmentAPI多环境
npm install
npm run build       # 并发构建 client / ssr / edge
npm run start:ssr   # 跑 ssr 产物：打印 “Hello from node vXX”
npm run start:edge  # 跑 edge 产物：打印 “Hello from edge (Web/Worker runtime)”
```

观察点：

1. `npm run build` 终端同时出现三个环境的构建日志、产物分别进 `dist/client|ssr|edge`——**多环境**。
2. `start:ssr` 与 `start:edge` 跑的是同一份 `src/shared/app.js`，但 `runtime` 一个是 Node 版本、一个是边缘标识——**多运行时**（靠 `this.environment.name` 分流）。
3. 打开 `dist/ssr/entry-server.js` 和 `dist/edge/entry-edge.js` 对比：前者含 `process.version`，后者不含——证明两个环境**真的打进了不同的平台实现**。
4. 把 `builder.buildApp` 里的 `Promise.all` 改成顺序 `await`，重新 build，看日志顺序变化——亲手体会 `buildApp` 是「环境编排器」。

## 六、什么时候真的需要它（别为用而用）

Environment API 很强，但不是每个项目都需要把它显式摆出来：

- **纯前端 SPA**：你根本感知不到它的存在——默认就一个 client 环境，照常 `vite build` 即可。**不需要碰 `environments`**。
- **传统 SSR（只有 client + Node）**：Vite 已内置 client/ssr 两个环境，多数 SSR 框架（Nuxt/SvelteKit 等）替你封装好了，你一般也不用手写。
- **真正用得上**：当你要产出**第三个及以上目标**（边缘函数、RSC、多种 worker 运行时），或要写一个**需要区分环境行为的插件**时——这才是 Environment API 大显身手的场景。

> 选型提醒：把它当「需要时才掏出的重型工具」，而不是每个项目的标配。判断标准很简单——**你是否有「同一份代码要产出 ≥3 种运行目标」的真实需求**。

## 七、本节小结

- Environment API（Vite 6+）把「构建目标」抽象成一等公民「环境」：每个环境有独立的模块图、`resolve`、`build`。`client`/`ssr` 只是内置默认环境，`edge` 等可自定义。
- 落地两入口：`environments` 声明有哪些环境；`builder.buildApp` 编排 build 时怎么构建它们（可并发可顺序）。
- 多运行时靠「业务代码 import 抽象平台入口 + 插件在 `resolveId` 里读 `this.environment.name` 分流」实现「一份源码、多运行时」。
- `this.environment` 是插件感知「当前为哪个环境工作」的关键入口，取代了过去粗糙的全局 `ssr` 布尔。
- 插件能共享时，用一套配置加 `this.environment` 分流；平台插件差异大或副作用不可控时，按环境拆配置。
- 别为用而用：纯 SPA / 传统 SSR 多数感知不到它；真正需要它的是「同一份代码产出 ≥3 种运行目标」或写「区分环境的插件」。

## 八、可直接用于项目的 checklist

- [ ] 先判断是否真需要多环境：只有 client（SPA）或 client+ssr（传统 SSR）时，不必显式写 `environments`。
- [ ] 需要第三类目标（edge/rsc/worker）时，在 `environments` 里新增一个环境键，配好它的 `build.outDir` 与 `resolve.conditions`。
- [ ] 跨运行时差异用「抽象平台入口 + 按环境解析」隔离，避免在业务代码里散落 `if (ssr)`。
- [ ] 写插件要区分环境时，用 `this.environment.name`，而不是猜全局命令。
- [ ] 判断插件是否适合共享：行为差异小就用 `this.environment` 分流；依赖差异大、产物约束不同或副作用不可控就按环境拆配置。
- [ ] 用 `builder.buildApp` 统一编排多环境构建；环境间无依赖就 `Promise.all` 并发，有依赖就按序 `await`。
- [ ] 边缘环境记得 `resolve.conditions: ['worker', 'edge']` 且通常 `noExternal: true`（产物自包含）。
