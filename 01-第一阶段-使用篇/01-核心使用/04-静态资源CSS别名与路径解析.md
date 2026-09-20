# 04 · 静态资源、CSS（预处理器 / CSS Modules / PostCSS）、别名与路径解析

> 本节解决的真实工程问题：图片到底该 `import` 还是丢 `public/`？为什么有的图片打包后带了哈希、有的没有？Sass/Less 怎么接？CSS Modules 怎么用才不串样式？以及那个几乎人人踩过的坑——**别名在 dev 好好的，`tsc` 却报找不到模块**。本节把 Vite 的资源与样式处理规则讲清。

事实基线：2026 年中，Vite 8.x。

---

## 一、静态资源：import 还是 public，先分清

Vite 处理静态资源有两条完全不同的通道，选错就会出“线上图片 404”或“缓存不更新”。

### 通道 A：被代码 `import`（推荐，走处理流程）

```js
import logoUrl from './logo.png';
img.src = logoUrl; // 得到的是处理后的最终 URL
```

- dev 下返回可访问 URL；build 下文件被拷到 `dist/assets/` 并**加上内容哈希**（如 `logo.a1b2c3.png`）。
- 哈希 = 长效缓存友好：内容变了文件名才变，浏览器缓存能放心设很久。
- **小于阈值的资源会被内联成 base64 data URL**（默认 4KB，由 `build.assetsInlineLimit` 控制），省一次请求。

几种 import 变体：

```js
import url from './img.png';                 // 拿 URL（默认）
import raw from './shader.glsl?raw';         // 拿文件文本内容
import workerUrl from './worker.js?url';     // 强制拿 URL，不内联
import Worker from './worker.js?worker';     // 作为 Web Worker 构造器
```

动态/批量引入用 `import.meta.glob`：

```js
// 把某目录下所有图片一次性收集（懒加载，返回动态 import 函数）
const images = import.meta.glob('./icons/*.svg');
// 想直接拿 URL 字符串：
const urls = import.meta.glob('./icons/*.svg', { query: '?url', import: 'default', eager: true });
```

`import.meta.glob` 是 Vite 提供的编译期能力，不是浏览器原生 API。Vite 会在 dev/build 时把 glob 展开，具体结果取决于资源类型和参数：如果匹配的是 JS/组件模块，普通写法通常会转换成一组动态 `import()`；像上面这种静态资源配合 `?url` + `eager: true`，构建后会直接得到一组资源访问 URL。

### 通道 B：放 `public/`（原样拷贝，不处理）

- 用根绝对路径引用：`<img src="/banner.png">`，**不要 import**。
- 不加哈希、不内联、不被转换，构建时原样拷到产物根。
- **适用**：不需要被代码引用的固定资源（favicon、robots.txt、需要稳定 URL 的第三方文件）。
- **不适用**：业务图片/字体——放 public 会失去哈希带来的缓存优化，且改了文件名不变容易被旧缓存命中。

> 判断口诀：**能被 import 的，就 import**（拿哈希、拿缓存优化）；**必须保持固定 URL、或不该被处理的，才放 public**。

### 常见坑：CSS / JS 里别手写源码绝对路径

在 CSS 里 `background: url(./bg.png)` 是会被 Vite 处理的（解析成最终 URL）。但如果你拼字符串 `url('/src/assets/bg.png')` 这种绝对源码路径，build 后路径就错了——**让 Vite 解析，别手写源码路径**。

## 二、CSS：开箱即用的部分

Vite 对 CSS 的基础能力**零配置**：

- 直接 `import './style.css'`，样式自动注入页面，且支持 HMR。
- `@import`、`url()` 自动解析。
- build 时 CSS 默认被抽取成单独文件并压缩（库模式等场景可配 `build.cssCodeSplit`）。

### CSS 预处理器：装上依赖就行

Vite **内置了对 Sass/Less/Stylus 的支持，但不自带编译器**——你只需安装对应依赖，无需配 loader：

```bash
npm install -D sass        # .scss / .sass
# 或 npm install -D less / stylus
```

然后直接用：

```js
import './style.scss';
```

需要全局注入变量（每个文件自动 `@use 'variables'`）时，用 `css.preprocessorOptions`：

```ts
// vite.config.ts
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@use "@/styles/variables.scss" as *;`,
      },
    },
  },
});
```

### CSS Modules：防样式污染

文件名带 `.module.` 即开启 CSS Modules，类名被局部哈希化，杜绝全局串样式：

```css
/* button.module.css */
.primary { color: white; background: #646cff; }
```

```js
import styles from './button.module.css';
btn.className = styles.primary; // 实际类名类似 _primary_x1y2z
```

可用 `css.modules` 调整命名规则（如 `localsConvention: 'camelCaseOnly'` 让 `kebab-case` 类名以驼峰访问）。

### PostCSS：自动接管

只要项目根有 `postcss.config.js`（或在 `vite.config` 的 `css.postcss` 配置），Vite 会自动把它应用到所有 CSS。最常见的是 autoprefixer 加浏览器前缀：

```js
// postcss.config.js
export default {
  plugins: {
    autoprefixer: {},
  },
};
```

> 关系厘清：**预处理器（Sass）→ 把 `.scss` 编译成 CSS；PostCSS → 对 CSS 做后处理（加前缀、嵌套、未来语法降级）**。两者不冲突，常一起用：Sass 编译完的 CSS 再过 PostCSS。

## 三、别名与路径解析：dev 与 tsc 要“两边都配”

这是本节最高频的坑。别名让 import 更短：

```ts
// vite.config.ts
import { fileURLToPath, URL } from 'node:url';
export default defineConfig({
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
});
```

但**只配这一处不够**。`resolve.alias` 只负责 Vite 在编译/打包时把 `@/x` 解析成真实文件；它**管不到 TypeScript 的类型解析**。所以 `tsc` / IDE 仍会报 `Cannot find module '@/x'`。解决：在 `tsconfig.json` 同步配 `paths`：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

> 记住：**别名 = Vite 的 `resolve.alias` + tsconfig 的 `paths`，两份要一致。** 这正好呼应 [07 TypeScript 处理](./07-TypeScript处理.md) 里“Vite 只转译、类型检查交给 tsc”的事实——两件事两套配置。

`resolve.extensions` 控制可省略的扩展名（默认含 `.js/.ts/.jsx/.tsx/.json` 等）；`resolve.dedupe` 在 monorepo 里强制依赖去重（见第一阶段后续 monorepo 章节）。

## 四、动手：资源与样式全家桶

配套代码：[第一阶段-使用篇/01-核心使用/04-静态资源与CSS/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/01-核心使用/04-静态资源与CSS/)

这个 demo 一次性演示：import 资源（带哈希）vs public 资源、Sass + 全局变量注入、CSS Modules、PostCSS（autoprefixer）、`@` 别名、`import.meta.glob` 批量引入。

```bash
cd vite_brochure_code_public/第一阶段-使用篇/01-核心使用/04-静态资源与CSS
npm install
npm run dev
npm run build && npm run preview
```

观察点：
- `npm run build` 后看 `dist/assets/`：被 import 的图标带哈希，`public/` 里的文件在 `dist/` 根、无哈希。
- 看 CSS Modules 生成的类名被哈希化。
- 看 autoprefixer 给需要前缀的属性加了 `-webkit-` 等（取决于 `browserslist` 目标）。
- 删掉 `tsconfig.json` 的 `paths`，用 `npx tsc --noEmit` 看别名类型报错（dev 仍正常）——亲手验证“两边都要配”。

## 五、本节小结

- 资源两通道：能 import 就 import（拿哈希、可内联、缓存友好）；必须固定 URL 才放 `public/`（不处理、不哈希）。
- CSS 零配置可用；预处理器只需装依赖（`sass` 等），无需 loader；`.module.css` 开 CSS Modules 防污染；有 `postcss.config.js` 即自动接管。
- 别名要 **`resolve.alias` + tsconfig `paths` 两处同步**，否则 dev 正常但 `tsc`/IDE 报错。

## 六、可直接用于项目的 checklist

- [ ] 业务图片/字体走 `import` 拿哈希；只有 favicon/robots 等放 `public/` 并用 `/xxx` 引用。
- [ ] 需要内联的小图标确认在 `assetsInlineLimit` 阈值内（或按需调整）。
- [ ] 用了 Sass/Less 时已安装对应编译器依赖（`sass`/`less`）。
- [ ] 需要复用样式的项目用了 CSS Modules（`.module.css`）或方案明确，避免全局类名冲突。
- [ ] 需要浏览器前缀时配了 `postcss.config.js` + autoprefixer，并设置了 `browserslist`/`build.target`。
- [ ] 别名在 `vite.config` 与 `tsconfig.json` **两处**都配且一致。
- [ ] CSS/代码里引用资源用相对路径让 Vite 解析，没有手写 `/src/...` 源码绝对路径。
