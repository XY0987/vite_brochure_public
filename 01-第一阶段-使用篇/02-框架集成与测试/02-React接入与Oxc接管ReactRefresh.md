# 02 · React 接入与 Oxc 接管 React Refresh

> 本节解决的真实工程问题：用 Vite 8 写 React，为什么现在默认装 `@vitejs/plugin-react` 就够了？旧文章里常见的 `@vitejs/plugin-react-swc`、`@vitejs/plugin-react-oxc`、`react({ babel })` 还该不该用？为什么有时「改组件，输入框里的字没了/计数器归零」——Fast Refresh 没生效？本节把 React 插件这条「JSX 编译 + Fast Refresh」链路讲透，并给出失效排查清单。

事实基线：2026 年中，Vite 8.x（Rolldown 默认、Oxc 接管转译/压缩）。`@vitejs/plugin-react` 6.x 使用 Oxc 做 React Refresh transform，Babel 不再是默认依赖；`@vitejs/plugin-react-oxc` 已被合并进主包并 deprecated；React 19。

---

## 一、React 插件到底替你做了两件事

接着 [01 框架插件集成机制](./01-框架插件集成机制.md) 的结论，React 插件具体落地为两件事：

1. **JSX/TSX 编译**：把 `<App/>` 这种 JSX 编译成函数调用（`jsx()` / `React.createElement()`）。
2. **React Fast Refresh**：注入热更新接管代码，做到「改组件→局部更新→**保留组件 state**」。

你写的是这样：

```jsx
import { useState } from 'react';

export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>count is {n}</button>;
}
```

浏览器最终拿到的是编译后的标准 JS + 一段 Fast Refresh 样板。这两件事都由插件完成。

## 二、React 插件怎么选，以及 Vite 8 里「谁接管了转译」

这是 React + Vite 用户绕不开的选择题。Vite 8 后的结论比旧教程简单很多：**默认用官方 `@vitejs/plugin-react` v6**，因为它已经把 Oxc Fast Refresh 链路合进来了。

| 插件 | 底层编译器 | 在 Vite 8 的状态 | 适用 |
|---|---|---|---|
| `@vitejs/plugin-react` | Oxc（React Refresh transform）+ Vite/Rolldown 管线 | **官方默认，Vite 8 首选** | 绝大多数项目，默认就用它 |
| `@vitejs/plugin-react-swc` | SWC（Rust） | 仍可用于偏好 SWC 或历史项目 | 明确需要 SWC 行为时再选 |
| `@vitejs/plugin-react-oxc` | Oxc（Rust） | **已 deprecated，能力已合并进 `@vitejs/plugin-react`** | 不建议新项目使用 |

### 为什么说「Vite 8 用 Oxc 接管 React Refresh」

Vite 8 用 **Rolldown** 做唯一打包器、**Oxc** 接管转译与压缩（见 [核心使用](../01-核心使用/README.md) 与第二/三阶段）。React 这条链路也被卷进了这次「全面 Rust 化」：

- 在 Vite 8 里，官方 `@vitejs/plugin-react` v6 不再依赖 Babel 来做 React Refresh transform——这部分已由 Oxc 接管，安装体积也随之下降；
- 「`@vitejs/plugin-react` 用 Oxc 接管 React Refresh」的含义：**转译这件事的实际执行者，从 JS（Babel）让位给了 Rust（Oxc）体系**；
- 旧的 `@vitejs/plugin-react-oxc` 曾是过渡包，现在已被合并进主包并 deprecated。看到旧文章推荐它，先看发布时间。

> 一句话：**Vite 8 时代写 React，直接用官方 `@vitejs/plugin-react` 即可——它已经站在 Rust/Oxc 链路上**；除非有特定 SWC 诉求，才考虑 `-swc` 变体。

### 选型决策树

- 用 Vite 8、没有特殊诉求 → 官方 `@vitejs/plugin-react`（默认推荐，本节 demo 即用它）。
- 明确偏好 SWC 或历史项目已有 SWC 链路 → `@vitejs/plugin-react-swc`。
- 看到 `@vitejs/plugin-react-oxc` → 视为旧资料，Vite 8 新项目改用 `@vitejs/plugin-react`。

这些插件的最小配置入口接近，迁移成本通常很低——基本就是改一行 import 和 `react()` 调用。但 Vite 8 下不要再把 `@vitejs/plugin-react-oxc` 当成新包来引入。

## 三、配置长什么样

最小配置（官方插件）：

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
});
```

换成 SWC 版几乎只改 import：

```ts
import react from '@vitejs/plugin-react-swc';

export default defineConfig({
  plugins: [react()],
});
```

普通 React 项目一般不需要手动配置 Babel，`react()` 已经能处理 JSX 和 Fast Refresh。只有你还要额外跑一层 Babel 转换时，比如接入 React Compiler，才需要把 Babel 插件加进来。

在 Vite 8 / `@vitejs/plugin-react` v6 里，不再把 Babel 配置写进 `react({ babel })`（这个选项已经从类型里移除），而是把 Babel 当成一个独立插件，显式加 `@rolldown/plugin-babel`：

```ts
import react from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [
    react(),
    babel({
      plugins: [/* 例如 'babel-plugin-react-compiler' */],
    }),
  ],
});
```

> 顺序说明：官方 `@vitejs/plugin-react` v6 release notes 与 React 官方文档的写法是 `react()` 在前、`babel()` 在后；插件顺序会影响 transform 的先后，照官方写即可。接 React Compiler 时还可以用插件导出的 `reactCompilerPreset` 简化配置（`babel({ presets: [reactCompilerPreset()] })`）。
>
> 注意：`react()` 一定要放进 `plugins` 数组。「装了依赖但忘了加进 plugins」是新手最常见的「JSX 报解析错误」原因。

## 四、React Fast Refresh 原理（简单理解）

Fast Refresh 是 React 官方的热更新方案，Vite 的 React 插件把它接到了 Vite 的 `import.meta.hot` 通道上。它能做到「改组件、保留 state」，靠的是几条约定：

1. **以「文件」为热替换单位**，且要求文件里**导出的都是 React 组件**（函数组件 / 用 `memo`/`forwardRef` 包的组件）。
2. 插件给每个组件生成稳定签名。改动后，运行时用新组件**替换**旧组件的渲染逻辑，但**复用旧的 Hook 状态**（`useState` 的值不丢）。
3. 一旦插件判断「这次改动没法安全地只换组件」（比如文件里还导出了非组件的东西），就**放弃 Fast Refresh，退化为整页刷新**，state 清空。

这解释了那个高频困惑——**为什么我改个组件，输入框/计数器的值有时保留、有时清空**：保留=Fast Refresh 生效；清空=退化成整页刷新了。

## 五、Fast Refresh 失效排查清单

按命中概率从高到低：

### 1. 组件文件里混入了非组件导出（最常见）

```jsx
// ❌ 同一个文件里既导出组件又导出常量/函数 → Fast Refresh 可能退化
export default function Panel() { /* ... */ }
export const PANEL_WIDTH = 320;          // 这一行会破坏热替换
export function formatTitle(s) { /* */ } // 这也会
```

**解决**：组件文件只导出组件，把 `PANEL_WIDTH`、`formatTitle` 挪到 `constants.ts` / `utils.ts`。

### 2. 匿名默认导出

```jsx
export default () => <div/>;   // ❌ 匿名，拿不到稳定签名
```

**解决**：

```jsx
export default function Foo() { return <div/>; }  // ✅ 具名
```

### 3. 自定义 Hook 与组件混在一处、或组件不是「顶层函数」

Fast Refresh 对「组件定义在另一个函数内部」「条件式定义组件」等非常规写法支持有限。保持组件是模块顶层的具名函数。

### 4. 没装/没启用插件，或扩展名没覆盖

- `plugins: [react()]` 漏了；
- 用 `.jsx`/`.tsx` 之外的扩展名写 JSX（插件默认匹配 `.jsx`/`.tsx`）。

### 5. 改的是公共依赖文件

改 `store.ts`、`main.tsx` 这种被大量模块依赖的文件，HMR 沿模块图冒泡到根 → 整页刷新。这是机制使然，不是 bug。

## 六、动手：感受 Fast Refresh 保留状态

配套代码：[第一阶段-使用篇/02-框架集成与测试/02-React接入/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/02-框架集成与测试/02-React接入/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/02-框架集成与测试/02-React接入
npm install
npm run dev
```

观察点：

1. 页面有个计数器，先点几下让它变成非零。
2. 改 `src/Counter.jsx` 里的按钮文案并保存 → **文案更新、但计数不归零**（Fast Refresh 生效）。
3. 打开 `src/Counter.jsx`，**取消注释那行 `export const EXTRA = ...`**（一个非组件导出），再改文案保存 → 这次**计数归零**（退化为整页刷新）。亲眼验证第五节第 1 条。
4. demo 默认用官方 `@vitejs/plugin-react`（v6 起 React Refresh transform 走 Oxc）；`vite.config.ts` 里保留了 SWC 版的历史选项提示，方便你理解不同插件的边界。

## 七、本节小结

- React 插件做两件事：**JSX/TSX 编译** 和 **Fast Refresh（保留 state 的热更新）**。
- Vite 8 里官方 `@vitejs/plugin-react` v6 已用 Oxc 接管 React Refresh transform，默认用它即可；`@vitejs/plugin-react-oxc` 是过渡包，已 deprecated。
- Fast Refresh 以文件为单位、要求「只导出组件」，否则退化为整页刷新——这正是「state 有时保留有时清空」的原因。
- 失效排查从「非组件导出」「匿名导出」查起，命中率最高。

## 八、可直接用于项目的 checklist

- [ ] Vite 8 新项目写 React，默认选官方 `@vitejs/plugin-react`（React Refresh transform 已走 Oxc）；有 SWC 诉求再评估 `@vitejs/plugin-react-swc`，不要新装已 deprecated 的 `@vitejs/plugin-react-oxc`。
- [ ] 确认 `react()` 已加入 `vite.config` 的 `plugins`，JSX 用 `.jsx`/`.tsx` 扩展名。
- [ ] 组件文件「只导出组件」，常量/工具函数拆到独立文件，保住 Fast Refresh 边界。
- [ ] 默认导出用具名函数组件，避免匿名 `export default () => ...`。
- [ ] 改组件后 state 莫名清空时，先排查是否混入了非组件导出。
