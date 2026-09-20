# 03 · 选型指南：什么项目用什么、Vite 处于什么位置

> 本节解决的真实工程问题：知道了每个工具的定位（01 节）和它们的关系（02 节），落到一个具体项目上，你还是会卡在「**那我到底该用哪个**」——做一个后台管理系统用 Vite 还是 webpack？写一个开源工具库用 Rollup 还是 Vite 库模式？要个极速 CLI 打包用 esbuild 还是 Rolldown？本节给一套**按场景对号入座**的选型指南，并最终回答一个容易被忽略的问题：**Vite 自己在这张工具地图上到底站在哪个位置**。
>
> 事实基线：2026 年中，Vite 8.1.x / Rolldown 1.1.x / Oxc。
>
> 边界：本节是**使用 / 选型视角**的「怎么选」。各工具的实现级深度对比（模块图、产物差异根因）见第二阶段「专题 · 底层引擎与生态对比」。

> 前置：建议先读 [01](./01-五个工具各自的定位与擅长场景.md) 与 [02](./02-它们的关系-谁负责dev谁负责build谁在取代谁.md)，本节直接用它们建立的词汇。

---

## 一、先把一个认知误区掰正：Vite 不是打包器

这是选型的前提。很多人把 Vite 和 webpack / Rollup 摆在同一层比较「Vite vs Rollup 谁好」——**这是类目错误**。

- **Vite 是一个「构建工具 / 框架」**：它提供 dev server、HMR、配置体系、插件体系、对 TS/CSS/静态资源的开箱处理。
- **它自己不打包**——打包这件事它**委托给底层的打包器**（Vite 2–7 委托给 Rollup，Vite 8 委托给 Rolldown），转译/压缩委托给 Oxc。

所以 Vite 的真实身份是**「编排层 / 集成层」**：把打包器（Rolldown）+ 转译压缩（Oxc）+ dev server + 插件生态**整合成一套开箱即用的体验**。

> **一句话定位 Vite：它是站在打包器之上的「指挥」，不是打包器本身。** 「Vite vs Rollup」不该问谁好，而该问「我要直接用打包器（Rollup/Rolldown），还是用 Vite 帮我把打包器包好的整套方案」。

理解这点，下面的选型就顺了。

## 二、按场景选：6 类典型项目

### 1. 现代前端应用（SPA / MPA / SSR）—— 用 Vite

后台系统、官网、营销页、SSR 站点、各种业务应用：**默认 Vite**。

理由：你要的是「开发体验 + 开箱即用 + 合理产物」的整套方案，而不是自己拼打包器 + dev server + HMR。Vite 8 底层的 Rolldown+Oxc 已经把速度和产物都拉满了，你直接享受成果即可。**2026 年新起一个应用项目，没有特殊理由就是 Vite。**

### 2. 开源 / 内部 JS-TS 库 —— Rollup 系 或 Vite 库模式

发 npm 的库（要输出 esm/cjs/类型声明）：

- **追求产物最干净、配置最成熟** → 直接用 **Rollup**（或基于它的 `unbuild`）。库打包是 Rollup 的主场。
- **想少配置、和应用共用一套 Vite 心智** → 用 **Vite 库模式**（`build.lib`，底层 Rolldown）。[01 核心使用 / 06 SSR、库模式、多页](../01-核心使用/06-SSR库模式多页应用.md) 讲过用法。
- **只要极简 + 极快、能接受产物控制弱一点** → `tsup`（基于 esbuild）。

> 经验法则：**库越「基础、被大量项目依赖」，越倾向 Rollup（产物质量优先）；库越「内部、迭代快」，越可以 Vite 库模式或 tsup（效率优先）**。

### 3. 极速转译 / 简单打包（CLI、脚本、Node 工具）—— esbuild 系

写个 CLI、Node 服务打包、需要把 TS 飞快转成 JS：**esbuild / tsup**。这类场景对「产物分包精细度」要求低、对「速度」要求高，正是 esbuild 的甜区。用 Vite 反而是杀鸡用牛刀。

### 4. 微前端 / Module Federation —— 看历史包袱

- **历史上**：Module Federation 是 webpack 普及的，重度 MF 体系常年绑定 webpack。
- **现在**：Vite 8 + `@module-federation/vite` 已能跑通 MF（05 章 03 节有可运行 demo）。新项目做微前端可以用 Vite 方案。
- **决策**：已有 webpack MF 体系且稳定 → 不必为迁而迁；全新微前端 → 优先评估 Vite + MF 插件。

### 5. 重度依赖 webpack 专属生态的老项目 —— 谨慎评估迁移

如果项目深度绑定某些**只有 webpack 才有的 loader/plugin**，或构建逻辑高度定制：

- 别盲目迁 Vite。先评估这些 webpack 专属能力在 Vite 生态有没有等价替代。
- 真要迁，按 05 章「迁移案例」的方法拆解：配置迁移、插件替换、构建差异、回滚方案。
- 也可考虑 **Rspack**（Rust 重写的 webpack），它兼容 webpack 配置/生态，是「想要 Rust 速度又不想离开 webpack 生态」的折中。

### 6. 只想要极速 Lint —— Oxc（oxlint），与构建工具无关

`oxlint` 可以独立接入任何项目（哪怕你用 webpack），作为比 ESLint 快几十倍的检查器。这跟你用什么打包器**完全正交**，单独评估即可。

## 三、选型速查表


| 你的场景                    | 首选                                    | 备选                    | 不推荐                   |
| ----------------------- | ------------------------------------- | --------------------- | --------------------- |
| 前端应用（SPA/MPA/SSR）       | **Vite**（底层 Rolldown+Oxc）             | Rspack（重 webpack 生态时） | 裸用 Rollup/esbuild 自己搭 |
| 发布到 npm 的基础库            | **Rollup** / unbuild                  | Vite 库模式              | webpack（太重）           |
| 内部库、迭代快                 | **Vite 库模式** / tsup                   | Rollup                | webpack               |
| CLI / Node 工具 / 极速转译    | **esbuild / tsup**                    | Rolldown(直用)          | Vite（过重）              |
| 微前端 / Module Federation | **Vite + @module-federation/vite**（新） | webpack MF（存量）        | 自己手搓运行时               |
| 重度 webpack 老项目          | **维持现状 / 评估 Rspack**                  | 渐进迁 Vite（见 05 章）      | 一把梭重写                 |
| 极速代码检查                  | **oxlint（Oxc）**                       | ESLint                | —                     |




## 四、Vite 在工具地图上的位置（一张图）

```
┌──────────────────────────────────────────────┐
│                  应用开发者                      │
│            （你写业务代码的地方）                  │
└───────────────────────┬──────────────────────┘
                        │ 你直接面对的是这一层
                        ▼
┌──────────────────────────────────────────────┐
│        Vite（编排层 / 集成层 · 构建工具）          │
│  dev server · HMR · 配置体系 · 插件体系 · 开箱处理 │
└───────┬───────────────────────────┬──────────┘
        │ 把「打包」委托给           │ 把「转译/压缩」委托给
        ▼                           ▼
┌────────────────┐          ┌────────────────┐
│   Rolldown     │  内部调用 │      Oxc        │
│  （打包器·Rust）│◄────────►│ （工具链·Rust）  │
└────────────────┘          └────────────────┘
   同层友商：Rollup/webpack      同层友商：esbuild/terser/swc
   （Rolldown 取代 Rollup）      （Oxc 取代 esbuild 转译压缩）
```

读这张图的三个要点：

1. **Vite 在「编排层」**，不和 Rolldown/Rollup 同层——它是用打包器的，不是打包器。
2. **打包器层**里 Rolldown 取代了 Rollup 的位置；webpack 是这一层的「另一套全家桶」（自带编排，所以它既是打包器也是编排层，这也是它重的原因）。
3. **工具链层**里 Oxc 取代了 esbuild/terser 在 Vite 中的转译/压缩职责。

> 对比 webpack 你会更懂 Vite 的取舍：webpack 是「编排 + 打包」捏在一个大包里（强但重）；Vite 是「编排层薄、打包/转译外包给专精的 Rust 工具」（分层、可演进——正因为分层，它才能从 Rollup 平滑换到 Rolldown 而上层基本不变）。

## 五、动手：用对比 demo 给「选型」找手感

选型不能只靠记表格，跑一次对比能建立「速度/产物」的肌肉记忆。复用 01 节的三打包器对比 demo：

配套代码：[第一阶段-使用篇/06-打包工具横向对比/01-三打包器横向对比/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/06-打包工具横向对比/01-三打包器横向对比/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/06-打包工具横向对比/01-三打包器横向对比
npm install
npm run compare
```

把它和你的选型判断对照：

- esbuild 转译极快但产物控制弱 → 印证「CLI/极速转译选它，精细分包别指望它」。
- Rollup 产物稳但慢（小 demo 上就明显最慢）→ 印证「库打包选它，大型应用 dev 别用它」。
- Rolldown 速度对标原生工具、又有 Rollup 级产物控制 → 印证「应用开发交给 Vite（底层就是它）最省心」。

## 六、本节小结

- **Vite 不是打包器，是编排层**：它把打包（Rolldown）+ 转译压缩（Oxc）+ dev server + 插件整合成开箱方案。「Vite vs Rollup」是类目错误。
- 按场景选：**应用→Vite；基础库→Rollup；内部库→Vite 库模式/tsup；CLI/极速转译→esbuild；微前端→Vite+MF 插件或存量 webpack；重 webpack 老项目→维持或评估 Rspack；极速 lint→oxlint**。
- **Vite 的位置**：站在打包器之上的指挥层；正因分层，它能从 Rollup 平滑换到 Rolldown 而上层不变——这正是 webpack「编排+打包一体」做不到的灵活。
- 选型原则：先判断「要整套方案还是要单点工具」，再按场景对号入座，别为「新」而选。

## 七、可直接用于项目的 checklist

- [ ] 起新项目前先问：我要「整套构建方案（Vite）」还是「单点打包器（Rollup/esbuild）」？
- [ ] 应用类项目默认 Vite，没有特殊理由不要自己拼打包器 + dev server。
- [ ] 发 npm 的基础库优先 Rollup（产物质量）；内部/快速迭代库可用 Vite 库模式或 tsup。
- [ ] CLI / Node 工具 / 纯转译选 esbuild 系，别上 Vite。
- [ ] 微前端：新项目评估 Vite + `@module-federation/vite`；存量 webpack MF 稳定就别硬迁。
- [ ] 重度依赖 webpack 专属生态时，先确认 Vite 有等价替代再决定迁移，必要时考虑 Rspack。
- [ ] 永远记住：选 Vite ≈ 选「Rolldown + Oxc + 一套好用编排」，所以底层引擎的优点你都吃得到。