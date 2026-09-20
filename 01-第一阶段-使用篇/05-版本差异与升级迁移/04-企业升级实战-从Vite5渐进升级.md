# 04 · 企业升级实战：从 Vite 5 渐进式升级到 8 与风险评估

> 本节解决的真实工程问题：个人 demo 升级一把梭就行，但**企业项目不一样**——几十个依赖、一堆自研/社区插件、CI 卡点、线上用户。你不能某天直接把 `vite: ^5` 改成 `^8` 然后祈祷构建能过。一旦升级后产物出问题、某个插件静默失效、线上白屏，回滚成本极高。本节给一套**渐进式、可回滚、风险可控**的升级路线：从 Vite 5 出发，先升 Node 和小版本，再用 `rolldown-vite` 作为「隔离层」在不改 `vite` 依赖的前提下试水 Rolldown，最后正式升到 Vite 8。配套一个可运行的「升级体检脚本」，帮你在动手前把迁移热点圈出来。

事实基线：2026 年中，目标版本 Vite 8.1.x。`rolldown-vite` 作为 drop-in 隔离层是 Vite 7 时代的过渡产物，本节讲清它的原理与「在升级流程中的位置」。

> 前置：[02 配置迁移](./02-各版本升级要点与配置迁移.md) 讲了「具体改哪些配置」，本节讲「按什么顺序、用什么策略改，才让企业项目升得稳」。两节配合使用。

---

## 一、企业升级的核心原则：把「一次大跳」拆成「几次小步」

直接从 Vite 5 跳到 Vite 8，等于**同时**承担：Node 升级 + 废弃特性清理 + 打包器换血（Rollup→Rolldown）+ 插件兼容 + CJS 行为变化。五件事一起上，出问题时你根本分不清是哪一环。

渐进式升级的精髓就一句话：

> **每一步只改一个变量，每一步都能独立验证、独立回滚。** 把「五件事一起赌」拆成「五次可控的小步」。

推荐的升级阶梯（从 Vite 5 出发）：

```
Step 0  体检：圈出迁移热点（不改代码）
Step 1  升 Node 到 20.19+/22.12+（不动 vite）
Step 2  Vite 5 → 7：清废弃特性、跑通构建（仍是 esbuild+Rollup 双引擎）
Step 3  用 rolldown-vite 隔离层试水 Rolldown（不改 vite 依赖、可秒回滚）
Step 4  Vite 7 → 8：正式切到默认 Rolldown，改配置名、验证产物
Step 5  清理：移除隔离层、关掉过渡开关、对比线上指标
```

注意 Step 3 是关键的「风险隔离阀」——它让你在**不正式升级 Vite 大版本**的情况下，先验证「我的项目在 Rolldown 上跑得通吗」。下面重点讲它。

## 二、Step 0：升级体检——先圈热点，再动手

升级前最该做的不是改代码，而是**盘清楚要改什么**。手动翻几十个配置文件不现实，写个脚本扫一遍最快。

配套 demo 提供了一个可运行的「升级体检脚本」，用静态文本扫描找出 `vite.config.*` 里的过时写法：

配套代码：[第一阶段-使用篇/05-版本差异与升级迁移/04-rolldown-vite隔离层/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/05-版本差异与升级迁移/04-rolldown-vite隔离层/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/04-rolldown-vite隔离层
npm install
npm run audit    # 扫描本目录的 vite.config，列出迁移项
```

输出示例：

```
=== Vite 8 升级体检 ===
📄 .../vite.config.js
  [提示] build.rollupOptions 是 rolldownOptions 的 deprecated 别名
         建议：重命名为 build.rolldownOptions（语义一致，可直接改名）
  [提示] output.manualChunks 是 Rollup 风格的函数式分包
         建议：迁移到声明式的 output.codeSplitting: { groups: [{ name, test }] }
共发现 2 处迁移项。
```

这个脚本本身很简单（一张「正则规则表」逐条匹配配置文本），但思路对企业很有用：**把团队踩过的迁移坑沉淀成规则，让脚本在升级前自动圈出来**，而不是靠人肉记忆。你可以把它接进 CI，作为升级前的 checklist 自动化。

> 体检只圈「确定要改的热点」，不替代真实构建验证。真正确认要靠后面的「隔离层 + 实跑产物对比」。

## 三、Step 3 详解：`rolldown-vite` 隔离层的原理与用法

这是整个升级流程里最该理解透的一环。

**问题**：你想知道「我的项目换成 Rolldown 后会不会出问题」，但又不想正式把 `vite` 升到大版本（大版本升级牵扯太多、回滚麻烦）。

`rolldown-vite` **的解法**：它是一个**与 Vite 同 API 的发行版**，内部把打包器换成了 Rolldown。你不改任何业务代码、不改 `vite.config`，只在 `package.json` 里用**包别名**把 `vite` 指向 `rolldown-vite`：

```jsonc
// package.json —— 用 overrides 把 vite 这个名字「重定向」到 rolldown-vite
{
  "overrides": {
    "vite": "npm:rolldown-vite@7.3.1"   // 锁定版本，保证可复现（配套 demo 见 package.rolldown-vite.example.json）
  }
}
// pnpm 用 "pnpm.overrides"，yarn 用 "resolutions"，语义相同
```

加上这段、重装依赖后，项目里所有 `import ... from 'vite'`、所有 `vite build` 命令，**实际跑的都是 Rolldown 版本**——而你的代码一行没动。

它为什么叫「隔离层 / drop-in」：

- **drop-in（无缝替换）**：API 兼容，业务代码零改动就能换引擎。
- **隔离层**：它把「换引擎」这件事隔离成「一段可增删的 `overrides` 配置」。验证通过就保留、出问题就**删掉这段、重装依赖，秒回到原引擎**——回滚成本极低。

**用它做什么**：在不正式升 Vite 大版本的前提下，提前暴露「我的项目 + 插件在 Rolldown 上的兼容性问题」。常见暴露出来的问题：某个 esbuild 插件失效、某个依赖 Rollup 内部实现的插件报错、CJS 依赖导入行为变化。**这些问题你迟早要面对，隔离层让你提前、低风险地面对。**

> 时间线提醒：`rolldown-vite` 是 **Vite 7 时代**的过渡方案——那时默认引擎还是 esbuild+Rollup，隔离层用来「提前试 Rolldown」。到了 **Vite 8**，Rolldown 已是默认引擎，隔离层的「试水」使命基本完成。所以这一步主要服务于「现在还在 Vite 5/6/7、想稳妥升 8」的团队：先用隔离层验证 Rolldown 兼容性，验证通过再正式升 8。

## 四、风险评估：升级前必须想清楚的几件事

渐进式只是流程，**风险评估**才是企业升级的核心动作。逐项评估：


| 风险维度       | 怎么评估                                        | 缓解手段                                     |
| ---------- | ------------------------------------------- | ---------------------------------------- |
| 插件兼容性      | 列出所有插件，标注「官方/社区/自研」「是否依赖 Rollup 内部/esbuild」 | 隔离层提前试跑；失效的找替代或升级版本                      |
| 产物正确性      | 升级前后对比 chunk 数量、体积、入口、关键页面运行                | 用 manifest + 体积报告 diff，而非「构建没报错就行」       |
| CJS/ESM 行为 | 排查项目里 `require`/混用 CJS 依赖的地方                | 出问题先用 `legacy.inconsistentCjsInterop` 定位 |
| 构建环境       | Node 版本、CI 镜像、Docker 基础镜像                   | 先单独升 Node 这一步                            |
| 回滚预案       | 出问题能不能快速退回？退回点在哪                            | 每步独立提交；隔离层可秒删；保留上一版构建产物                  |


> 风险评估的黄金标准：**「升级后线上出问题，我能在多久内回滚到已知可用状态？」** 如果答案是「不确定」，说明你的升级步子太大了，回去拆得更细。

## 五、动手：体检 + 隔离层思路串一遍

配套 demo（`04-rolldown-vite隔离层/`）的 `vite.config.js` 故意保留了 `rollupOptions + manualChunks` 两处老写法：

```bash
cd vite_brochure_code_public/第一阶段-使用篇/05-版本差异与升级迁移/04-rolldown-vite隔离层
npm install
npm run audit    # 体检：圈出 2 处迁移项
npm run build    # 证明：老写法在 Vite 8 兼容层下仍能正常构建
```

观察点：

1. `npm run audit` 圈出 `rollupOptions` 和 `manualChunks` 两处——这就是「Step 0 体检」的产物。
2. `npm run build` 仍能成功——证明「先靠兼容层让项目跑起来」是可行的第一步，不必一上来就改全部配置。
3. 打开 `package.rolldown-vite.example.json`，对照第三节的 `overrides` 写法，体会「隔离层 = 一段可增删的依赖重定向」（demo 的 `package.json` 保持 `vite@8` 以便直接跑）。
4. 按体检结果把 `rollupOptions→rolldownOptions`、`manualChunks→codeSplitting` 改掉，再跑一次 `audit`，看迁移项归零——这就是「逐项消项」的迁移节奏。

## 六、本节小结

- 企业升级的核心是「把一次大跳拆成几次可独立验证、可独立回滚的小步」，而不是直接 `^5`→`^8`。
- 推荐阶梯：体检 → 升 Node → 升到 7 清废弃 → `rolldown-vite` 隔离层试水 Rolldown → 正式升 8 改配置 → 清理对比。
- `rolldown-vite` 是 drop-in 隔离层：用 `package.json` 的 `overrides`/`resolutions` 把 `vite` 别名到它，零改代码换引擎，出问题删掉即回滚。它是 Vite 7 时代「提前试 Rolldown」的工具，服务于「想稳妥升 8」的团队。
- 风险评估按维度逐项做：插件兼容、产物正确性、CJS 行为、构建环境、回滚预案。黄金标准是「出问题多久能回滚到已知可用状态」。
- 升级体检脚本把团队踩过的坑沉淀成规则，可接 CI，在动手前自动圈热点。

## 七、可直接用于项目的 checklist

- [ ] 升级前先跑体检脚本（或等价手段），把所有 deprecated 写法、风险插件列成清单。
- [ ] 每一步只改一个变量（先只升 Node，再只清废弃，再只换引擎……），每步独立提交。
- [ ] 用 `rolldown-vite` 隔离层在「不升 Vite 大版本」的前提下先验证 Rolldown 兼容性。
- [ ] 验证不通过时，删掉 `overrides` 段、重装依赖即回滚，定位具体是哪个插件/依赖的问题。
- [ ] 逐个核对插件：官方/社区/自研，是否依赖 Rollup 内部或 esbuild。
- [ ] 用「产物 diff」（chunk 数、体积、入口、关键页面）验收，而不是「构建没报错」。
- [ ] 明确每一步的回滚点，确保「出问题能快速退回已知可用状态」。
- [ ] 升级完成后移除隔离层、关掉 `legacy.*` 过渡开关，并对比线上性能/体积指标。