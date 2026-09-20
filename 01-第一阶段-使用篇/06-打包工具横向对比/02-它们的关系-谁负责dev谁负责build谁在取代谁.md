# 02 · 它们的关系：谁负责 dev、谁负责 build、谁在取代谁

> 本节解决的真实工程问题：上一节给每个工具立了身份，但工程里更让人迷糊的是它们**怎么串起来**——为什么 Vite 早年要同时用 esbuild 和 Rollup？「Rolldown 取代 Rollup」「Oxc 取代 esbuild」这些说法到底取代的是哪部分？webpack 又是被谁取代的？本节用「**dev/build 分工**」和「**取代时间线**」两条线，把这 5 个工具的关系一次理清，让你读 Vite 8 的任何架构图都不再卡壳。
>
> 事实基线：2026 年中，Vite 8.1.x / Rolldown 1.1.x / Oxc。

> 前置：建议先读 [01 五个工具各自的定位与擅长场景](./01-五个工具各自的定位与擅长场景.md)，本节默认你已分清「打包器 vs 工具链」。

---

## 一、第一条线：dev 和 build 是两件事

要理解这几个工具的关系，先接受一个事实：**前端构建有两个差别巨大的场景**，对工具的要求几乎相反。

- **dev（开发服务器）**：要的是「快速冷启动 + 快速热更新」。你改一行代码，希望浏览器立刻反映。这里**最怕慢**，对产物质量没要求（反正不部署）。
- **build（生产构建）**：要的是「产物又小又对、tree-shaking 干净、分包合理」。这里**最怕产物烂**，慢一点可以忍（CI 里跑一次）。

这两个相反的诉求，正是 Vite 早年「双引擎」的根源——它干脆**用两个不同的工具分别伺候 dev 和 build**。

## 二、Vite 的双引擎时代（Vite 2–7）


| 场景    | 用谁          | 干什么                                         |
| ----- | ----------- | ------------------------------------------- |
| dev   | **esbuild** | ① 依赖预构建（把 node_modules 预打包成 ESM）② 转译 TS/JSX |
| build | **Rollup**  | 生产打包：tree-shaking、分包、产物优化                   |


这套组合很聪明：dev 用最快的 esbuild 保证开发体验，build 用产物最优的 Rollup 保证上线质量。**各取所长**。

但代价是「**两套引擎 = 两套模块图 = 两套行为**」，于是出现 Vite 长期被诟病的问题：

> **dev/production 行为不一致**：开发时 esbuild 这么处理、构建时 Rollup 那么处理，某些边界情况（CommonJS 互操作、CSS 处理、tree-shaking 差异）会导致「dev 跑得好好的，build 出来挂了」的诡异 bug。这类 bug 的根因，就是两个引擎对同一份代码理解不完全一致。

这是理解后面所有演进的「病因」。Vite 6/7/8 的主线之一，就是**治这个病**。

## 三、第二条线：取代关系（谁在替谁）

Vite 6 起两条独立主线并行推进（05 章详述），落到「工具取代」上，关系是这样的：

```
                  ┌─────────────── Vite 8（单引擎）───────────────┐
   双引擎时代      │                                              │
                  │   dev  ─┐                                     │
  esbuild(dev) ───┼──────►  ├─►  Rolldown（统一 dev + build 的打包）│
  Rollup(build) ──┼──────►  ┘                                     │
                  │                                              │
  esbuild(转译) ──┼──────────────►  Oxc（转译 + 压缩）              │
  terser(压缩) ───┼──────────────►  Oxc                           │
                  └──────────────────────────────────────────────┘
```

拆开说三组取代：

1. **Rolldown 取代 Rollup（build）+ esbuild 的「打包/预构建」职责**。dev 不再用 esbuild 打包、build 不再用 Rollup，二者收敛到 Rolldown 一个 Rust 引擎。**双引擎 → 单引擎**，dev/build 行为不一致的病根被切掉。
2. **Oxc 取代 esbuild 的「转译」职责**。TS/JSX → JS 这一步从 esbuild（Go 实现）换成 Oxc（Rust 实现，属 Vite 自家可控的工具链，与 Rolldown 同源）。
3. **Oxc 取代 terser/esbuild 的「压缩」职责**。生产压缩也交给 Oxc。

> **关键澄清**：esbuild 在 Vite 里是被「**拆开取代**」的——它的「打包」给了 Rolldown，它的「转译/压缩」给了 Oxc。不是某一个工具一对一替掉它。这就是为什么 01 节非要先分清「打包器 vs 工具链」。



## 四、Rolldown 和 Oxc 不是竞争，是搭档

新手常误以为「Rolldown 和 Oxc 是两个互相竞争的 Rust 打包器」。**错**。它俩是分工搭档：

- **Rolldown** 负责「打包」——组织模块图、tree-shaking、分包、产出 chunk。
- **Oxc** 负责「单文件处理」——解析、转译、压缩。
- **Rolldown 内部就调用 Oxc** 来做解析和转译。可以理解为：Oxc 是 Rolldown（以及整个新 Vite 栈）的「Rust 零件库」。

所以 Vite 8 的底层是一套 **Rust 组合拳**：`Rolldown（骨架/打包）+ Oxc（零件/转译压缩）`，而不是「Rolldown vs Oxc 二选一」。

## 五、webpack 是被谁取代的？

前面四节都在讲 Vite 内部的工具更替，那 webpack 呢？它不在 Vite 内部，关系要单独说：

- webpack 是**整个 Vite 方案**在「现代前端应用构建」这条赛道上取代的对象，而不是被某个单一工具点对点替掉。
- 取代它的不是「esbuild」或「Rollup」单兵，而是 **Vite 这套「dev 用原生 ESM 按需编译 + build 用打包器」的整体范式**（第一阶段 01 节讲过这个范式之争）。
- webpack 自己也在反击（Rspack——Rust 重写的 webpack），所以更准确的说法是：**「JS 打包器」这一代正在被「Rust 打包器」整体取代**，Vite/Rolldown 和 Rspack 是这股浪潮里的两支。

> 一句话定位：**Vite（范式）取代 webpack（范式）；Rolldown（Rust）取代 Rollup（JS）；Oxc（Rust）取代 esbuild/terser（转译压缩）。** 三句话说清所有取代关系。

## 六、独立使用时它们仍各自为政

别因为「在 Vite 里被取代」就以为这些工具死了。**脱离 Vite，它们各有活法**：

- **esbuild**：`tsup`、`tsx`、众多 CLI 打包仍以它为核心，独立场景里它依旧是「最快转译/打包器」之一。
- **Rollup**：海量 npm 库仍用 Rollup 直接打包（产物干净、配置成熟）。Rolldown 想完全接棒还需要生态时间。
- **webpack**：现役大型企业项目、强依赖其专属 loader/plugin 的场景仍在用。
- **Oxc**：`oxlint` 作为独立的超快 linter 单独流行，跟用不用 Vite 无关。

「在 Vite 内被谁取代」和「在生态里是否还活着」是两个问题，别混。

## 七、动手：亲眼确认 Vite 8 的 dev/build 都是 Rolldown

口说无凭，跑个探针看 Vite 8 实际把 dev 和 build 都交给了 Rolldown（而非历史上的 esbuild/Rollup 分治）。

配套代码：[第一阶段-使用篇/06-打包工具横向对比/03-Vite定位探针/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/06-打包工具横向对比/03-Vite定位探针/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/06-打包工具横向对比/03-Vite定位探针
npm install
npm run info     # 打印 Vite 内部各引擎真实版本（含 rolldownVersion）
npm run build    # 探针打印 command=build → 打包器：Rolldown
npm run dev      # 探针打印 command=serve → 打包器：Rolldown（与 build 同一引擎）
```

观察点：

1. `npm run info` 输出里 `rolldownVersion` 有值——证明 Rolldown 是真打包器；`rollupVersion` 仍存在但只是**兼容层**，不是主力。
2. `build` 和 `dev` 两次都打印出 `Rolldown`——这就是「**单引擎**」：dev/build 行为不一致的病根（双引擎）已经被消除。
3. 对比 05 章的「版本探针」demo，你会发现这是同一类探针——本章只是把它从「版本视角」换成「工具关系视角」来读。

## 八、本节小结

- 前端构建有 **dev / build** 两个诉求相反的场景：dev 怕慢、build 怕产物烂。
- **Vite 2–7 双引擎**：dev=esbuild（预构建+转译）、build=Rollup（产物优化）；代价是「两套引擎 → dev/build 行为不一致」的老 bug。
- **三组取代**：① Rolldown 取代 Rollup(build) + esbuild 的打包/预构建；② Oxc 取代 esbuild 的转译；③ Oxc 取代 terser/esbuild 的压缩。
- **Rolldown 与 Oxc 是搭档不是竞品**：Rolldown 打包、Oxc 处理单文件，Rolldown 内部用 Oxc。
- **webpack 被「Vite 范式」整体取代**，而非被单一工具点替；同时「JS 打包器一代」正被「Rust 打包器一代」整体替换（Rolldown / Rspack）。
- 这些工具脱离 Vite 仍各自活跃（esbuild→tsup、Rollup→库打包、oxlint 等）。



## 九、可直接用于判断的 checklist

- [ ] 看到「X 取代 Y」，先问取代的是「打包」还是「转译/压缩」职责——避免把 esbuild 的多重身份混为一谈。
- [ ] 记住 Vite 8 底层 = `Rolldown（打包）+ Oxc（转译/压缩）`，二者搭档而非竞争。
- [ ] 排查「dev 正常、build 异常」类问题时，知道这是历史「双引擎」的典型病征——Vite 8 单引擎后这类问题应大幅减少，若仍出现优先怀疑插件兼容。
- [ ] 评估 webpack 去留时，把它放进「JS 一代 vs Rust 一代」的大趋势里看，而不是只跟某个单一工具比。
- [ ] 选独立工具（非 Vite 项目）时，别因为「Vite 弃用了 esbuild」就否定 esbuild——独立场景它依然是顶级选择。