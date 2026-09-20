# 第一阶段 · 使用篇 / 02 框架集成与测试

> 本章对应《Vite：从使用到精通》第一阶段第 2 项「框架集成与测试」。
> 事实基线：2026 年中，Vite 8.x（Rolldown 1.0 已于 2026.05 稳定，Vite 8 默认唯一打包器，Oxc 接管转译/压缩）。

本章目标：把 Vite「核心使用」的能力接到真实技术栈上——理解框架插件到底替你做了什么（不是装上就完事），出问题（HMR 不生效、JSX 报错、SFC 不热更）时知道是哪一环；并掌握与 Vite 同源的测试方案 Vitest，让「跑得起来」和「测得了」在同一套配置里闭环。

## 章节目录

| 序号 | 标题 | 解决的真实问题 | 配套代码 |
|---|---|---|---|
| 01 | [框架插件集成机制](./01-框架插件集成机制.md) | 框架插件到底接管了哪几步、HMR 为什么靠它 | `01-框架插件集成机制/` |
| 02 | [React 接入与 Oxc 接管 React Refresh](./02-React接入与Oxc接管ReactRefresh.md) | Vite 8 为什么默认用 `@vitejs/plugin-react` v6，旧 SWC/Oxc/Babel 写法怎么判断，Fast Refresh 如何排查 | `02-React接入/` |
| 03 | [Vue 与 Svelte 接入](./03-Vue与Svelte接入.md) | SFC / `.svelte` 单文件组件是怎么被编译进 Vite 管线的 | `03-Vue与Svelte接入/` |
| 04 | [Vitest：与 Vite 共享配置与转换管线](./04-Vitest原生测试.md) | 为什么测试不用再单独配一套 Babel/ts-jest，配置如何复用 | `04-Vitest测试/` |

## 这一章和「核心使用」的关系

01 章讲的是 Vite 的「裸」能力：dev server、按需编译、`resolveId → load → transform`、HMR 通道。本章不重复这些，而是回答一个新问题：**框架是怎么搭在这套能力之上的**。

- 一句话主线：框架插件 = 在 Vite 的 `transform` 钩子里把 `.vue` / `.jsx` / `.svelte` 编译成浏览器能跑的 JS，并往里注入 HMR 代码；Vitest = 复用同一条 `transform` 管线去跑测试。
- 所以本章反复用到的概念（`transform` 钩子、`import.meta.hot`、模块图）都来自 01 章，建议先读完 [01 核心使用](../01-核心使用/README.md)。

## 配套代码

所有可运行 demo 位于配套 GitHub 仓库 [第一阶段-使用篇/02-框架集成与测试/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/02-框架集成与测试/)，按上表「配套代码」列一一对应。每个 demo 自包含、可独立 `npm install` 后运行。运行环境要求（Node ≥ 20.19 / 22.12）见代码仓库根 README。

## 阅读建议

01 先建立「框架插件做了什么」的心智模型，再按你用的技术栈选读 02（React）或 03（Vue/Svelte）——两章是并列关系，可只读其一。04 Vitest 与框架无关，是所有人都绕不开的实战内容，建议必读。
