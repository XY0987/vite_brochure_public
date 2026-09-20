# 02 · monorepo / workspace 下的 Vite 组织与依赖处理

> 本节解决的真实工程问题：项目长大后，迟早会从「一个仓库一个应用」演进成「一个仓库多个应用 + 一堆内部共享包」的 monorepo。这时一连串怪事开始冒出来：改了内部包 `@company/ui` 的源码，应用端**死活不热更**，非得重新 `build` 内部包甚至重启；明明只装了一个 React，控制台却报 `Invalid hook call`、`两个 React 实例`；dev 启动时报 `The request url is outside of Vite serving allow list`；某个内部包一引入 dev 就报 504 或解析失败。这些**几乎全是 monorepo 特有的依赖与解析问题**，跟你单仓库时的经验对不上。本节把 monorepo 下 Vite 的组织方式和最常见的几类依赖坑（软链、内部包热更新、重复依赖、依赖外部化）一次讲清，给你一套「新建 monorepo 该怎么配 Vite」的可复制方案。

事实基线：2026 年中，Vite 8.x。下文以 npm workspaces 为例，pnpm / yarn workspace 原理一致，差异单独标注。

---

## 一、workspace 的本质：node_modules 里的一条软链

先把最关键的机制讲透，后面所有坑都从它派生。

monorepo 一般长这样（三种包管理器都支持 `workspaces` 概念）：

```
my-monorepo/
├── package.json          # 根：声明 workspaces
├── packages/
│   └── ui/               # 内部包 @company/ui
└── apps/
    └── web/              # 应用，依赖 @company/ui
```

> 说明：下文用 `@company/ui` 作为内部包的**通用占位名**；配套 demo 里这个包的实际名字是 `@demo/ui`（见观察点），两者指同一回事。

根 `package.json` 声明工作区：

```json
{
  "private": true,
  "workspaces": ["packages/*", "apps/*"]
}
```

在根目录跑一次 `npm install`，包管理器做的关键动作是：**在 `node_modules/@company/ui` 建一条软链（symlink），指向 `packages/ui`**。

```bash
node_modules/@company/ui -> ../../packages/ui
```

于是应用里 `import '@company/ui'` 时，顺着这条软链拿到的是 **`packages/ui` 的真实源码目录**，而不是一份拷贝。**这条软链就是「内部包能被实时消费」的物理基础**——也是后面所有 HMR / 重复依赖问题的根源。

> pnpm 差异：pnpm 默认用「软链 + 全局 store 硬链」的更严格结构，内部包同样是软链；它对「幽灵依赖」管得更死（没在 `package.json` 声明的依赖默认 import 不到），这点对 Vite 影响后面会提。

## 二、组织方式：应用各自一份 vite.config，内部包不打包

monorepo 下一个常见纠结：内部包要不要自己配 Vite、要不要先 build 成产物？

**推荐心智：应用（apps/*）才是 Vite 的「构建单元」，每个应用一份自己的 `vite.config`；内部库包（packages/*）默认不预先打包，直接以源码形式被应用消费。**

原因很实在：

- 内部包**先 build 再被引用**，意味着你每次改内部包都要重新构建它、应用才能看到新代码——dev 体验直接退回石器时代，还彻底丧失跨包 HMR。
- 让应用直接吃内部包的**源码**，Vite 就能把内部包的模块纳入自己的模块图，享受按需编译、HMR、统一的 TS/别名处理——**这才是 monorepo 用 Vite 的最大红利**。

要让「应用吃内部包源码」成立，内部包的 `package.json` 必须把入口指向源码：

```json
// packages/ui/package.json
{
  "name": "@company/ui",
  "type": "module",
  "exports": {
    ".": "./src/index.js"     // ← 指向源码，不是 dist/index.js
  }
}
```

> 这是 monorepo 里「内部包改了不热更」最高频的根因：内部包 `exports` 指向的是 **打包产物**（`dist/`），应用拿到的是死的构建结果，改源码当然不触发任何更新。开发期让 `exports` 指向源码即可。（若内部包还要对外发布 npm，可用 `publishConfig` 或条件导出区分「开发指向源码、发布指向产物」。）

## 三、坑一：内部包热更新（HMR）失效

现象：改 `packages/ui/src/Button.js`，应用页面纹丝不动，得手动刷新或重启。

排查顺序（从最常见到最少见）：

1. **内部包 `exports`/`main` 是不是指向了 `dist` 产物？** → 改成指向 `src` 源码（见上一节）。这是 90% 的原因。
2. **内部包源码是不是 CJS？** Vite 的 HMR 建立在 ESM 之上，内部包用 `module.exports` 写的 CJS 会被当依赖预构建、丧失 HMR。→ 内部包用 ESM（`"type": "module"` + `export`）。
3. **dev server 没权限读到内部包文件**（报 `outside of Vite serving allow list`）→ 见坑四的 `server.fs.allow`。

确认内部包以源码 ESM 形式被消费后，HMR 就跨包生效了：你改内部包的组件，应用页面热更新，和改应用自己的代码毫无差别。

## 四、坑二：重复依赖（duplicate deps）与 dedupe

现象：`Invalid hook call` / `You might have more than one copy of React` / 某个单例库（状态管理、i18n、emotion）行为异常。

根因：monorepo 下，应用和多个内部包可能各自在自己的 `package.json` 里依赖了 `react`，包管理器在某些情况下没把它们提升（hoist）成同一份，导致 node_modules 里**存在多份 react**。运行时多份 react 实例互不认账，hooks、context 全乱。

Vite 的解药是 `resolve.dedupe`：强制指定的依赖在整个应用里**只解析成同一份**。

```js
// apps/web/vite.config.js
export default defineConfig({
  resolve: {
    // 把容易冲突的「单例型」核心依赖列进来强制去重
    dedupe: ['react', 'react-dom'],
  },
});
```

> 经验：`dedupe` 名单里放的是那些「全局必须唯一」的库——`react`/`react-dom`、`vue`、状态管理、`@emotion/*`、`styled-components` 等。普通工具库（lodash 这类无状态的）多份并存顶多胖一点，不至于报错，不用都塞进去。
> pnpm 补充：pnpm 可用 `.npmrc` 的 `dedupe-peer-dependents` 或 `overrides`/`resolutions` 从包管理器层面收敛版本，和 Vite 的 `resolve.dedupe` 是两层手段，可叠加。

## 五、坑三：依赖外部化与预构建边界

monorepo 下还有两个和「依赖怎么被处理」相关的配置要心里有数：

### `optimizeDeps`：内部包不要被预构建

Vite 启动 dev 时会对 `node_modules` 里的依赖做**预构建**（pre-bundling，见 [01 核心使用 / 依赖预构建](../01-核心使用/05-依赖预构建.md)）。但内部 workspace 包是你正在开发、要热更的，**不能**被预构建成一坨不可热更的产物。

好消息：Vite 默认会自动把「软链进来的 workspace 包」排除出预构建。但当你遇到内部包被错误预构建（改了不更新、或预构建报错）时，显式声明意图更稳：

```js
optimizeDeps: {
  exclude: ['@company/ui'],   // 内部包用源码 + HMR，别预构建
},
```

反过来，如果某个内部包**体积大、又很少改**（比如一个稳定的图标库），你也可以反其道用 `optimizeDeps.include` 把它纳入预构建换取 dev 启动速度——按需权衡。

### `ssr.noExternal`：SSR 下内部包要打进包

如果应用是 SSR 的，还要注意：SSR 构建默认把 `node_modules` 里的依赖**外部化**（交给 Node 运行时 `require`）。但内部 workspace 包往往是 ESM 源码、还可能 import 了 `.css`/用了别名，被外部化后 Node 直接加载会炸（`ERR_REQUIRE_ESM` / 语法错误）。这时要把内部包标记为 `noExternal`，让 Vite 把它打进 SSR 包：

```js
ssr: {
  noExternal: ['@company/ui'],   // 内部包打进 SSR 包，而非外部化
},
```

> SSR + monorepo 的外部化坑，本章 [03 节 SSR 工程坑位](./03-SSR工程坑位排查手册.md) 会再展开。这里先建立「内部包在 SSR 下常需 noExternal」的条件反射。

## 六、坑四：dev server 的文件访问白名单

现象：dev 启动后访问内部包文件报：

```
The request url "/.../packages/ui/src/Button.js" is outside of Vite serving allow list.
```

根因：Vite 出于安全，dev server 默认只允许访问**项目根以内**的文件。monorepo 里内部包在 `packages/` 下、跟应用是平级目录，落在应用根之外，于是被拦。

解法：把 workspace 根放进白名单：

```js
// apps/web/vite.config.js
server: {
  fs: {
    allow: ['../..'],   // 从 apps/web 往上两级到 monorepo 根
  },
},
```

> 多数情况下 Vite 能自动探测 workspace 根并放行，不用手配；但目录层级特殊、或用了非标准 workspace 结构时，手动加 `server.fs.allow` 是稳妥兜底。

## 七、动手：一个能跨包热更的最小 monorepo

配套 demo 用 npm workspaces 搭了「一个应用 + 一个内部包」，你能亲手验证软链、跨包 HMR，以及上面几个配置的作用。

配套代码：[第一阶段-使用篇/04-多环境与工程化场景/02-monorepo工作区/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/04-多环境与工程化场景/02-monorepo工作区/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/04-多环境与工程化场景/02-monorepo工作区
npm install      # 在 monorepo 根执行，自动建软链
npm run dev      # 启动应用 @demo/web
```

观察点：

1. `npm install` 后看 `node_modules/@demo/ui` 是一条**软链**指向 `packages/ui`——workspace 的物理本质。
2. 应用用 `import { createButton } from '@demo/ui'`（按包名，不是相对路径）就能跑起来。
3. **不重启**，改 `packages/ui/src/Button.js` 的文字保存——页面按钮立刻更新（**跨包 HMR**）。
4. 把 `packages/ui/package.json` 的 `exports` 改成指向一个打包产物，再试——改源码不再热更，**亲手复现「内部包改了不生效」**。
5. 看 `apps/web/vite.config.js` 里的 `server.fs.allow` / `resolve.dedupe` / `optimizeDeps.exclude` 三件套注释，对照本节理解每个的用途。

## 八、本节小结

- workspace 的本质是 `node_modules` 里一条指向内部包源码的**软链**；monorepo 下 Vite 的所有依赖坑都从这条软链派生。
- 组织原则：应用是构建单元、各自一份 `vite.config`；内部库包不预先打包，`exports` 指向**源码**，让应用直接吃源码、享受按需编译与 HMR。
- 内部包热更失效，九成是 `exports` 指向了 `dist` 产物或内部包是 CJS——改成指向 ESM 源码即可。
- 重复依赖（`Invalid hook call` 等）用 `resolve.dedupe` 强制单例型核心依赖只解析一份。
- 依赖处理边界三件套：`optimizeDeps.exclude` 让内部包不被预构建（保 HMR）、SSR 下用 `ssr.noExternal` 把内部包打进包、`server.fs.allow` 放行 workspace 根。

## 九、可直接用于项目的 checklist

- [ ] 内部库包 `package.json` 的 `exports`/`main` 开发期指向**源码**（`src/index.js`），不是 `dist`。
- [ ] 内部包统一用 ESM（`"type": "module"` + `export`），别用 CJS，否则丧失 HMR。
- [ ] 把 `react`/`vue`/状态管理等「单例型」核心依赖放进应用的 `resolve.dedupe`，避免多实例报错。
- [ ] 内部包改了不热更：先查 `exports` 是否指向产物、是否 CJS、dev 是否报 allow list 错误。
- [ ] SSR 应用把内部 workspace 包加进 `ssr.noExternal`，避免外部化导致的 `ERR_REQUIRE_ESM`。
- [ ] 遇到 `outside of Vite serving allow list`，配 `server.fs.allow` 放行到 monorepo 根。
- [ ] 用 pnpm 时注意「幽灵依赖」：每个包用到的依赖都要在自己 `package.json` 里显式声明。
