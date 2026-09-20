# 04 · Vitest：与 Vite 共享配置与转换管线的原生测试

> 本节解决的真实工程问题：用 Jest 时，明明 Vite 已经能把 TS/JSX/别名/CSS 处理得好好的，却还要为测试再单独配一套 `babel.config`、`ts-jest`、`moduleNameMapper`、`transform`——两套配置经常不一致，「dev 能跑、测试报错」。为什么 Vitest 能省掉这一切？它和 Vite 到底共享了什么？本节讲清 Vitest 与 Vite 的「同源」关系，并给出从零接入与排查的实战路径。

事实基线：2026 年中，Vitest 4.x，与 Vite 8.x 共用同一套解析/转换管线。

---

## 一、Vitest 凭什么「不用再配一套」

一句话：**Vitest 直接复用你项目里的 `vite.config`——同一套插件、同一套 `resolve.alias`、同一条 `transform` 管线。** 它不是「另一个测试框架恰好支持 Vite」，而是**建在 Vite 之上**的测试运行器。

对比一下传统 Jest 的痛点：

| 关注点 | Jest（传统） | Vitest |
|---|---|---|
| TS/JSX 转译 | 另配 `ts-jest` / `babel-jest` | 复用 Vite 的 transform（Oxc/插件） |
| 路径别名 `@/` | 另写 `moduleNameMapper` | 复用 `resolve.alias` |
| CSS/静态资源 import | 另配 `transform`/mock | 复用 Vite 处理（或一键 mock） |
| 框架（Vue/React） | 另配 preset | 复用 `@vitejs/plugin-*` |
| 配置真实来源 | 与 dev 构建**两套，易漂移** | 与 dev 构建**同一套** |

Vitest 之所以在 Vite 生态里绕不开，核心就在于它「与 Vite 共享配置与转换管线、生态耦合最紧」：**你为 dev/build 配好的东西，测试里自动生效，不存在「两套配置不一致」的经典坑。**

> 所以掌握了前面 01–03 章（插件、别名、转换），你其实已经把 Vitest 的「环境」配好了一大半——这就是「同源」的红利。

## 二、最小接入：三步

### 1. 安装

```bash
npm i -D vitest
```

### 2. 在 `vite.config` 里加 `test` 字段

Vitest 复用 `vite.config`，测试配置就写在同一个文件的 `test` 字段里（需要类型提示时加一行三斜线引用）：

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite';

export default defineConfig({
  // plugins / resolve.alias 等会被测试自动复用
  test: {
    globals: true,          // 允许直接用 describe/it/expect，不用每次 import
    environment: 'node',    // 默认 node；测 DOM 用 'jsdom' 或 'happy-dom'
  },
});
```

> 关键点：**没有第二个配置文件**。`plugins`、`alias` 写一次，dev、build、test 三处共用。

### 3. 加脚本、写测试

```jsonc
// package.json
{
  "scripts": {
    "test": "vitest",          // watch 模式
    "test:run": "vitest run"   // 单次跑（CI 用）
  }
}
```

```ts
// src/math.test.ts
import { describe, it, expect } from 'vitest';
import { add } from './math';

describe('add', () => {
  it('两数相加', () => {
    expect(add(1, 2)).toBe(3);
  });
});
```

`npm test` 即可，TS、别名全自动生效——**因为它们就是 Vite 那套**。

## 三、测试环境：node / jsdom / happy-dom

纯逻辑（工具函数、store）用默认 `node` 即可。要测**操作 DOM 的代码**（组件、`document.querySelector`），得切换到浏览器环境模拟：

```ts
test: {
  environment: 'jsdom',     // 或 'happy-dom'（更快更轻，API 覆盖略少）
}
```

- `jsdom`：最成熟、API 全，体积/速度一般。
- `happy-dom`：更快更轻，绝大多数场景够用。

也可在单个测试文件顶部用注释覆盖环境：

```ts
// @vitest-environment jsdom
```

## 四、和框架插件配合（Vue/React 组件测试）

因为 Vitest 复用 `plugins`，组件测试几乎零额外配置：你 `vite.config` 里有 `vue()` / `react()`，测试里就能直接 import `.vue` / `.jsx` 组件。配合 Testing Library 这类库即可断言渲染结果：

```ts
// @vitest-environment jsdom
import { render, screen } from '@testing-library/vue';
import Counter from './Counter.vue';

it('点击后计数 +1', async () => {
  render(Counter);
  // ...触发点击、断言文本
});
```

> 关键体感：**dev 能正常渲染的组件，测试里就能 import 跑**，不需要像 Jest 那样再为 Vue/React 配 preset。这是「共享转换管线」最直接的好处。

## 五、共享配置带来的「副作用」与注意点

「共享」是优点，但也意味着**改 `vite.config` 会同时影响测试**，需要心里有数：

1. **别名改了，测试也跟着变**：好处是一致；但若给某环境单独加了 alias，要确认测试期望也一致。
2. **插件在测试期一样会跑**：极少数插件在 Node 测试环境下行为不同，可用 `test` 里的字段或条件区分。
3. **想给测试单开一份配置**：可以用单独的 `vitest.config.ts`（会与 `vite.config` 合并/覆盖），适合「测试需要不同 alias/插件」的复杂场景。但绝大多数项目**一份 `vite.config` 足够**，不要过早拆分。
4. **覆盖率**：`vitest run --coverage`（需装 `@vitest/coverage-v8`），同样跑在这套管线上。

## 六、动手：在一个项目里跑 Vitest

配套代码：[第一阶段-使用篇/02-框架集成与测试/04-Vitest测试/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/02-框架集成与测试/04-Vitest测试/)

```bash
cd vite_brochure_code_public/第一阶段-使用篇/02-框架集成与测试/04-Vitest测试
npm install
npm run test:run     # 单次跑全部测试
npm run test         # watch 模式，改源码自动重跑
```

观察点：

1. 测试 `src/math.test.ts` 直接 `import { add } from '@/math'`，用的是 `vite.config.ts` 里 `resolve.alias` 配的**路径别名 `@/`**——**别名零额外配置就生效**，这就是共享 `vite.config` 的证明。
2. `src/dom.test.ts` 顶部用 `// @vitest-environment jsdom` 指令把**本文件**切到 jsdom（`vite.config.ts` 里 `test.environment` 默认是 `node`），测一段操作 `document` 的代码，体会按文件切换环境。
3. 改 `src/math.ts` 让某个测试失败，看 watch 模式秒级反馈；改回即恢复。
4. 对比思考：若用 Jest，上面的别名和 TS 都得另配一套——而这里一行都没多写。

## 七、本节小结

- Vitest 是**建在 Vite 之上**的测试运行器：复用同一份 `vite.config`、同一套插件、`resolve.alias` 和 `transform` 管线。
- 接入只需三步：装 `vitest` → `vite.config` 加 `test` 字段 → 写脚本和测试。无需第二套 Babel/ts-jest/别名映射。
- 测 DOM 切 `environment: 'jsdom'`/`'happy-dom'`；测组件直接复用框架插件，几乎零配置。
- 「共享」是双刃剑：改 `vite.config` 会影响测试，需要时可用 `vitest.config.ts` 单开，但多数项目一份配置足矣。

## 八、可直接用于项目的 checklist

- [ ] 用 Vite 的项目，测试优先选 Vitest，避免维护两套（dev / test）配置。
- [ ] 测试配置写进 `vite.config` 的 `test` 字段，并加 `/// <reference types="vitest/config" />` 拿类型。
- [ ] 纯逻辑用 `environment: 'node'`，测 DOM/组件切 `jsdom` 或 `happy-dom`。
- [ ] 路径别名只在 `resolve.alias` 配一次，确认测试里直接可用（不要再写 `moduleNameMapper`）。
- [ ] CI 用 `vitest run`（非 watch）；需要覆盖率加 `--coverage` 与 `@vitest/coverage-v8`。
