# 03 · Vue 与 Svelte 接入：单文件组件是怎么进 Vite 管线的

> 本节解决的真实工程问题：`.vue` 文件里 `<template>`/`<script>`/`<style>` 三块混在一起，浏览器明明不认识，为什么 Vite 能跑？`<style scoped>` 的样式隔离、Vue 的 HMR、Svelte 的「编译期框架」特性，分别是插件在 Vite 哪一环做的？本节用「同一套集成机制、不同编译器」的视角，把 Vue 和 Svelte 一起讲清，让你换框架时心里有底。

事实基线：2026 年中，Vite 8.x。`@vitejs/plugin-vue` 6.x + Vue 3.5；`@sveltejs/vite-plugin-svelte` 7.x + Svelte 5。

---

## 一、同一套机制：编译 + HMR 注入

[01 框架插件集成机制](./01-框架插件集成机制.md) 的结论对 Vue/Svelte 同样成立——插件在 `transform` 钩子里把单文件组件编译成标准 JS 并注入 HMR。Vue 和 Svelte 的差异只在「编译器不同、编译产物形态不同」：

| 维度 | Vue | Svelte |
|---|---|---|
| 插件 | `@vitejs/plugin-vue` | `@sveltejs/vite-plugin-svelte` |
| 编译器 | `@vue/compiler-sfc` | `svelte/compiler` |
| 文件 | `.vue`（template/script/style 三块） | `.svelte`（markup/script/style） |
| 运行时 | 有运行时（虚拟 DOM / 响应式系统） | **几乎无运行时**，编译成直接操作 DOM 的命令式代码 |
| `<style>` 隔离 | `scoped` 加属性选择器 | 默认组件级作用域 |

> 记住这句话：**框架的「集成方式」是一样的（都是 transform 里编译 + 注入 HMR），不一样的是「编译成什么」**。Vue 编译成「render 函数 + 响应式」，Svelte 编译成「精简的命令式 DOM 代码」。这也是 Svelte 「没有运行时、产物小」卖点的根源——活儿在编译期干完了。

## 二、Vue 接入

### 配置

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [vue()],
});
```

### 一个 `.vue` 被拆成了什么

```vue
<!-- App.vue -->
<template>
  <button @click="n++">count is {{ n }}</button>
</template>

<script setup>
import { ref } from 'vue';
const n = ref(0);
</script>

<style scoped>
button { color: teal; }
</style>
```

`@vitejs/plugin-vue` 在 `transform` 里把它拆成三部分：

1. **`<template>` → render 函数**：`@vue/compiler-sfc` 编译成返回虚拟 DOM 的函数。
2. **`<script setup>` → 组件选项对象**：编译成标准 JS。
3. **`<style scoped>` → 独立 CSS 模块**：抽离出来，**走 Vite 的 CSS 管线**（见 [04 静态资源与 CSS](../01-核心使用/04-静态资源CSS别名与路径解析.md)），并给模板元素加上一个唯一属性（如 `data-v-xxxx`），用属性选择器实现 `scoped` 隔离。

所以浏览器看到的不是 `.vue`，而是「一个标准 JS 模块 + 一段被 Vite 注入的 CSS」。

> `<style scoped>` 的隔离不是魔法：插件给本组件每个元素加 `data-v-hash` 属性，把 `button {}` 改写成 `button[data-v-hash] {}`。理解这点，遇到「scoped 样式没生效 / 想穿透改子组件样式」时就知道为什么需要 `:deep()`。

### Vue 的 HMR

插件给每个 SFC 注入 `import.meta.hot.accept`，做到改 `<template>`/`<style>` 时局部更新、保留组件状态。Vue 的状态保留能力较强（响应式系统配合），但同样遵循「改公共文件会冒泡到整页刷新」的规律。

## 三、Svelte 接入

### 配置

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte()],
});
```

Svelte 还有一个惯例：项目根放一个 `svelte.config.js`，插件会读它（用于 preprocess、编译选项等）。即使是空配置也建议保留这个文件。

### 一个 `.svelte` 被编译成什么

```svelte
<!-- App.svelte -->
<script>
  let n = $state(0);   // Svelte 5 的 runes 语法
</script>

<button onclick={() => n++}>count is {n}</button>

<style>
  button { color: orangered; }
</style>
```

`svelte/compiler` 把它编译成**直接创建/更新 DOM 的命令式 JS**——没有虚拟 DOM、几乎没有运行时框架代码。`<style>` 默认就是组件级作用域（同样走 Vite CSS 管线）。这就是 Svelte「产物小、运行快」的来源：框架的活儿在编译期做完，运行时只剩你这点组件逻辑。

### Svelte 的 HMR

`@sveltejs/vite-plugin-svelte` 同样注入 HMR 接管代码，改组件局部更新。Svelte 的 HMR 在保留状态上也有自己的取舍（顶层 `$state` 一般能保留），机制与 Vue/React 同源——都是挂在 Vite 的 `import.meta.hot` 上。

## 四、三框架横向对比（使用视角）

| | React | Vue | Svelte |
|---|---|---|---|
| 插件 | `@vitejs/plugin-react`（Vite 8 默认 Oxc Fast Refresh） | `@vitejs/plugin-vue` | `@sveltejs/vite-plugin-svelte` |
| 写法 | JSX（JS 里写标签） | SFC（三块分明） | SFC 风格 + 编译期魔法 |
| 编译器 | Oxc/Babel/SWC | `@vue/compiler-sfc` | `svelte/compiler` |
| 运行时大小 | 较大（React+ReactDOM） | 中等 | **最小（编译期消化）** |
| 额外约定文件 | 无 | 无（可选 `jsconfig`） | 建议 `svelte.config.js` |

> 对「使用者」而言，三者接 Vite 的心智完全一致：**装插件 → 放进 `plugins` → 开写**。差异是各自语言/编译产物层面的，不影响你对 Vite 的使用方式。这正是 Vite「插件化集成框架」设计的好处——换框架，换插件，主干不变。

## 五、常见集成问题

1. **`.vue`/`.svelte` 报 “Failed to parse source” / 解析错误** → 插件没装或没加进 `plugins`（和 React 同理）。
2. **Vue `<style scoped>` 样式没作用到子组件** → scoped 只隔离本组件，穿透要用 `:deep()`。不是 Vite 的问题，是 scoped 机制。
3. **Svelte 改了不热更 / 报配置错** → 检查是否缺 `svelte.config.js`，以及插件版本是否匹配 Svelte 5。
4. **类型/IDE 报错但能跑** → `.vue`/`.svelte` 的类型靠各自的 IDE 插件（Volar / Svelte 扩展）与 `vue-tsc` 等工具，和 Vite「只转译不检查」一致（见 [07 TypeScript 处理](../01-核心使用/07-TypeScript处理.md)）。

## 六、动手：Vue 与 Svelte 并排跑

配套代码：[第一阶段-使用篇/02-框架集成与测试/03-Vue与Svelte接入/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/02-框架集成与测试/03-Vue与Svelte接入/)，内含两个独立子项目 `vue/` 和 `svelte/`。

```bash
# Vue
cd vite_brochure_code_public/第一阶段-使用篇/02-框架集成与测试/03-Vue与Svelte接入/vue
npm install && npm run dev

# Svelte（另开一个终端）
cd vite_brochure_code_public/第一阶段-使用篇/02-框架集成与测试/03-Vue与Svelte接入/svelte
npm install && npm run dev
```

观察点：

1. 两个项目都有计数器，先点几下，再改组件文案保存 → 局部更新、计数保留（各自插件的 HMR）。
2. `npm run build` 后对比 `dist/` 产物体积：Svelte 的 JS 通常**明显更小**（无运行时），直观感受「编译期框架」。
3. 改 Vue 的 `<style scoped>` 颜色、Svelte 的 `<style>` 颜色，观察样式被 Vite 注入、且作用域被隔离。

## 七、本节小结

- Vue/Svelte 接 Vite 用的是和 React **完全相同的集成机制**：`transform` 里编译 + 注入 HMR，只是编译器和产物形态不同。
- Vue：`@vue/compiler-sfc` 把 SFC 拆成 render 函数 + 组件对象 + 抽离的 scoped CSS（`data-v-hash` 属性选择器隔离）。
- Svelte：`svelte/compiler` 编译成命令式 DOM 代码，几乎无运行时，产物最小。
- 三框架对使用者的心智一致：装插件、放进 `plugins`、开写；换框架不影响你用 Vite 的方式。

## 八、可直接用于项目的 checklist

- [ ] Vue 项目：装 `@vitejs/plugin-vue` 并加入 `plugins`；想穿透 scoped 样式用 `:deep()`。
- [ ] Svelte 项目：装 `@sveltejs/vite-plugin-svelte`，保留 `svelte.config.js`，插件版本匹配 Svelte 5。
- [ ] `.vue`/`.svelte` 解析报错先查插件是否启用（与 React 同一套排查思路）。
- [ ] 类型检查交给 `vue-tsc` / IDE 扩展，别指望 `vite build` 报类型错（见 07 节）。
- [ ] 关注产物体积时，记得 Svelte 无运行时这一结构性差异，可用 `build` 产物对比佐证选型。
