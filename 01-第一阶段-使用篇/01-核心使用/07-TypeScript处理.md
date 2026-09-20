# 07 · TypeScript 处理：Vite 只转译、不做类型检查的事实与配套方案

> 本节解决的真实工程问题：明明类型写错了，`vite build` 却照样构建成功、还能上线，类型错误一个没拦住——这是 bug 还是设计？答案是**设计**。理解“Vite 只转译、不做类型检查”这个事实，并补上配套的类型卡点，是每个 TS + Vite 项目都必须做的事。

事实基线：2026 年中，Vite 8.x（转译由 Oxc 接管，速度极快；类型检查依然不在 Vite 职责内）。

---

## 一、核心事实：转译 ≠ 类型检查

Vite 处理 `.ts`/`.tsx` 时，做的是 **transpile-only（仅转译）**：把 TypeScript 语法**剥掉类型注解、转成浏览器能跑的 JavaScript**，仅此而已。它**不会**做类型检查（type checking）。

```ts
const n: number = 'I am a string'; // 类型明显错误
export const value = n;
```

这段代码 `vite build` **会成功**，产物里就是 `const n = 'I am a string'`。Vite 根本没去判断 `string` 能不能赋给 `number`。

### 为什么这么设计？因为速度

类型检查是**全程序**的：要把所有文件、所有类型关系都加载进来做推断，代价高、慢。而 Vite 的卖点就是快——dev 期按需转译单个文件、build 期由 Oxc 极速转译。如果每次转译都顺带做全量类型检查，Vite 的速度优势就没了。

所以 Vite（以及它底层用的 esbuild/Oxc/swc 这类工具）一致选择：**转译和类型检查分离**。转译只关心“语法对不对、能不能转成 JS”，类型检查交给专门的工具（`tsc`）在另外的环节做。

> 这呼应了第一阶段 [04 别名](./04-静态资源CSS别名与路径解析.md) 里的现象：Vite 的 `resolve.alias` 管编译解析，TS 的 `paths` 管类型解析——因为本来就是两套系统在干两件事。

### 一个直接后果：`isolatedModules` 限制

Vite dev 期处理 TS 时，核心心智是“拿到一个文件，就先把这个文件转成 JS 返回给浏览器”。它不会像 `tsc` 那样先看完整个项目的类型关系，所以有些依赖**跨文件类型信息**才能安全转译的写法，在 Vite 这类单文件转译工具里就容易出问题，典型是 `const enum` 和某些纯类型导入/再导出。

对应的实践是两条：

- `tsconfig.json` 开 `"isolatedModules": true`，让 `tsc` 按“每个文件都必须能被单独转译”的规则提前检查，把 Vite 转译阶段可能踩的坑先暴露出来。
- 纯类型导入/导出写成 `import type { Foo }` / `export type { Foo }`，明确告诉转译器“这只是类型，不是运行时代码”，生成 JS 时可以直接删掉；如果开启 `verbatimModuleSyntax`，这个边界会更严格。

## 二、配套方案：把类型检查补回来

既然 Vite 不查类型，你必须在别处查。三道防线，从开发期到 CI：

### 防线 1：IDE（实时，开发期）

VS Code 内置 TS 语言服务，写代码时就标红类型错误。这是第一道、也是反馈最快的防线。但它**只检查你打开/引用到的文件**，不保证全量，且不能阻止提交。

### 防线 2：`tsc --noEmit`（全量，本地/提交前）

用 TypeScript 编译器只做检查、不产出文件：

```bash
tsc --noEmit          # 普通 TS 项目
vue-tsc --noEmit      # Vue 项目（处理 .vue 单文件组件里的类型）
```

`--noEmit` 表示“只检查类型，别生成 JS”（JS 由 Vite 产出）。这是把全量类型检查和 Vite 转译分工的标准姿势：**Vite 出 JS，tsc 把关类型。**

把它固化进脚本：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "typecheck": "tsc --noEmit"
  }
}
```

注意这里的 `build` 脚本：**先 `tsc --noEmit` 再 `vite build`**。这样类型不过就不会进入构建，等于给 `vite build` 补上了类型卡点。Vue 项目同理用 `vue-tsc --noEmit && vite build`。

### 防线 3：CI 卡点（强制，合并前）

本地脚本可以被绕过（有人直接 `vite build`）。真正可靠的是在 CI 里加一步类型检查，类型不过就让流水线失败、阻止合并：

```yaml
# 示例：CI 里独立一步
- name: Type Check
  run: npm run typecheck   # = tsc --noEmit
```

> 为什么要独立成一步而不是只靠 `build` 脚本？因为 CI 里清晰的 `typecheck` 步骤失败信息更好定位，也能和单测、lint 并行跑，加快反馈。

## 三、几个 TS + Vite 的实务点

### 1. `vite/client` 类型

要让 `import.meta.env`、`import xxx from './a.png'`、`?raw`/`?worker` 等 Vite 特有写法有类型，需要引入 Vite 的客户端类型。新项目脚手架会生成 `src/vite-env.d.ts`：

```ts
/// <reference types="vite/client" />
```

### 2. 给 `import.meta.env` 自定义变量补类型

默认 `import.meta.env.VITE_FOO` 是 `string | undefined` 的宽泛类型。可以声明更精确的类型：

```ts
// src/vite-env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE: string;
  readonly VITE_ENABLE_DEBUG?: 'true' | 'false';
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### 3. 库项目要单独产出 `.d.ts`

Vite 库模式（见 [06](./06-SSR库模式多页应用.md)）只产 JS，不产类型声明。要让你的库有类型提示，用 `vite-plugin-dts` 或 `tsc --declaration --emitDeclarationOnly` 单独生成 `.d.ts`。

### 4. `tsconfig` 推荐基线

```jsonc
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler", // Vite 项目推荐
    "strict": true,
    "isolatedModules": true,        // 配合单文件转译
    "verbatimModuleSyntax": true,   // 类型导入/导出更明确
    "noEmit": true,                 // 由 Vite 产 JS，tsc 只检查
    "skipLibCheck": true,
    "types": ["vite/client"]
  }
}
```

## 四、动手：亲眼看到“类型错也能 build”

配套代码：[第一阶段-使用篇/01-核心使用/07-TypeScript处理/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/01-核心使用/07-TypeScript处理/)

demo 里**故意写了一个类型错误**，让你对比两条路径：

```bash
cd vite_brochure_code_public/第一阶段-使用篇/01-核心使用/07-TypeScript处理
npm install

# 路径 A：只让 Vite 转译——构建成功！类型错误被无视
npm run build:unsafe

# 路径 B：先 tsc 检查再构建——类型检查在这里拦下错误
npm run typecheck      # 报类型错误，退出码非 0
npm run build          # = tsc --noEmit && vite build，类型不过则中止
```

把 `src/buggy.ts` 里的类型错误修正后，`npm run build` 才会通过。这就直观证明了：**Vite 的成功 ≠ 类型正确，类型卡点必须自己加。**

## 五、本节小结

- Vite（底层 Oxc/esbuild）对 TS 是**仅转译**：剥类型、转 JS，不做类型检查——这是为速度做的设计取舍。
- 后果：`vite build` 成功不代表类型正确；`const enum` 等依赖跨文件类型的写法受限。
- 配套三道防线：IDE 实时 → `tsc --noEmit`（Vue 用 `vue-tsc`）全量 → CI 强制卡点。
- 把 `build` 脚本写成 `tsc --noEmit && vite build`，并在 CI 独立加 `typecheck` 步骤。
- 库项目额外用 `vite-plugin-dts`/`tsc` 产出 `.d.ts`。

## 六、可直接用于项目的 checklist

- [ ] 清楚 `vite build` 不查类型，团队不拿“build 通过”当类型正确的证据。
- [ ] `package.json` 里 `build` 脚本是 `tsc --noEmit && vite build`（Vue 用 `vue-tsc --noEmit`）。
- [ ] 有独立 `typecheck` 脚本，并在 CI 里作为强制卡点（类型不过阻止合并）。
- [ ] `tsconfig` 开了 `isolatedModules`，并按 Vite 推荐用 `moduleResolution: bundler`、`noEmit`。
- [ ] 项目引入了 `vite/client` 类型；对自定义 `VITE_*` 变量补了精确类型。
- [ ] 若发库，用 `vite-plugin-dts`/`tsc` 产出了 `.d.ts`。
