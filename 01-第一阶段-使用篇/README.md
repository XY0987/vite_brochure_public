# 第一阶段 · 使用篇

> 本阶段对应《Vite：从使用到精通》第一阶段「使用层面（用到精通）」。
> 事实基线：2026 年中，Vite 8.x（Rolldown 默认唯一打包器，Oxc 接管转译/压缩，Node 要求 20.19+ / 22.12+）。

第一阶段的目标不是让你“会启动一个 Vite 项目”，而是让你能在真实项目里独当一面：会配置、会集成框架和测试、会写/排插件、能处理 monorepo / SSR / 部署 / 版本升级，也知道 AI 给出的 Vite 答案该怎么校验。

## 模块地图

| 模块 | 解决的问题 | 建议读法 | 配套代码 |
|---|---|---|---|
| [01 核心使用](./01-核心使用/README.md) | 从定位、配置、命令、资源、预构建到 SSR/库/MPA、TS、性能与排障 | 必读基础；08/09 可作为排障手册反复查 | [01-核心使用/](https://github.com/XY0987/vite_brochure_code_public_public/tree/main/第一阶段-使用篇/01-核心使用/) |
| [02 框架集成与测试](./02-框架集成与测试/README.md) | React / Vue / Svelte 怎么接入，Vitest 如何复用 Vite 管线 | 用哪个框架就重点读对应节；Vitest 建议必读 | [02-框架集成与测试/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/02-框架集成与测试/) |
| [03 插件与生态](./03-插件与生态/README.md) | 插件怎么写、Rollup 钩子与 Vite 的关系、常用插件如何组合 | 想看懂配置里的 `plugins` 必读 | [03-插件与生态/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/03-插件与生态/) |
| [04 多环境与工程化场景](./04-多环境与工程化场景/README.md) | Environment API、monorepo、SSR 坑位、部署 CI 与缓存 | 团队项目、SSR、上线前重点读 | [04-多环境与工程化场景/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/04-多环境与工程化场景/) |
| [05 版本差异与升级迁移](./05-版本差异与升级迁移/README.md) | Vite 5→8 怎么理解和升级，配置名怎么迁移 | 升级/选型前先读 01→02→04 | [05-版本差异与升级迁移/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/05-版本差异与升级迁移/) |
| [06 打包工具横向对比](./06-打包工具横向对比/README.md) | esbuild / Rollup / Rolldown / webpack / Oxc 各自定位 | 做工具选型或解释技术方案时读 | [06-打包工具横向对比/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/06-打包工具横向对比/) |
| [07 AI 辅助 Vite 开发](./07-AI辅助Vite开发/README.md) | 如何让 AI 生成可维护配置、辅助排错，并用事实校验 | 第一阶段收口；建议在掌握前六章后读 | [07-AI辅助Vite开发/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/07-AI辅助Vite开发/) |

## 推荐阅读路径

顺序学习：`01 → 02 → 03 → 04 → 05 → 06 → 07`。这条路径从“单项目怎么用”推进到“团队工程怎么落地”，最后回到 AI 时代的使用方法。

按需跳读：

- 新项目落地：`01-02`、`01-03`、`01-04`、`02`、`07-01`。
- 构建慢 / 包太大 / 上线白屏：`01-08`、`04-04`、`05-02`。
- SSR / monorepo 工程问题：`01-06`、`04-02`、`04-03`。
- 从 Vite 5/6/7 升级：`05-01`、`05-02`、`05-04`。
- 让 AI 帮你但不被误导：先读 `05-02` 建立版本事实，再读 `07`。

## 术语基线

- Vite 8 推荐配置入口：`build.rolldownOptions`；`build.rollupOptions` 仍有兼容层，但属于旧名。
- Vite 8 推荐分包配置：`output.codeSplitting`；`manualChunks` 是 Rollup / Vite 5–7 写法，`advancedChunks` 是早期 Rolldown 过渡名，已 deprecated。
- React 新项目默认用 `@vitejs/plugin-react` v6；它已用 Oxc 接管 React Refresh transform。`@vitejs/plugin-react-oxc` 是过渡包，已 deprecated。
- Environment API 在 Vite 6 引入，仍应以官方文档和当前安装版本的类型/告警为准。

## 配套代码

配套代码托管在 GitHub：<https://github.com/XY0987/vite_brochure_code_public>（对应 `第一阶段-使用篇/` 目录）。

每个 demo 都尽量保持最小可运行。标准 demo 通常是：

```bash
npm install
npm run build
```

部分章节是多子项目或多终端场景（如 SSR、Vue/Svelte、Module Federation），请优先阅读对应 demo 目录下的 `README.md` 再运行。
