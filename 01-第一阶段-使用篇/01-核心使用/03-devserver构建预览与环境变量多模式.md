# 03 · dev / build / preview，与环境变量、多模式（mode）

> 本节解决的真实工程问题：`vite`、`vite build`、`vite preview` 三个命令到底各负责什么？为什么有人说“`preview` 通过不代表线上没问题”？还有最容易出事的一类 bug——**把后端密钥写进 `.env` 结果泄露到了浏览器**。本节把三命令的边界和环境变量的安全规则一次讲透。

事实基线：2026 年中，Vite 8.x。

---

## 一、三个命令，三种保证

| 命令 | 干什么 | 走哪条路径 | 保证什么 / 不保证什么 |
|---|---|---|---|
| `vite`（dev） | 启动开发服务器，按需编译 + HMR | dev 路径（Vite 8 用 Rolldown 的 dev 模式） | 保证开发体验快；**不**保证产物与线上一致 |
| `vite build` | 生产构建，完整打包优化 | build 路径 | 产出 `dist/`，做 tree-shaking/压缩/分包 |
| `vite preview` | 用静态服务器**预览 `dist/`** | 仅起一个静态服务器 | 验证“构建产物能不能跑”；**不**保证等同真实生产环境 |

三句话记住边界：

- **`dev` 不是生产**：它走按需编译路径，不打包。dev 正常 ≠ build 正常。
- **`preview` 不是生产**：它只是把 `dist/` 用一个简单静态服务器托起来，让你在本地验证“构建产物本身”有没有问题（比如路径、`base`）。它**没有**你线上的 Nginx 规则、CDN、网关、HTTP 头、SSR 运行时——所以 preview 通过只能排除“产物坏了”，不能替代上线前的真实环境验证。
- **真正的产物正确性看 `build`**：上线前必须跑 `build`，最好再 `preview` 看一眼。

> 这条“dev/build/preview 各保证什么”的边界，第二阶段会从 server 实现角度再讲一遍。用层面先建立纪律：**别用 dev 的表现替线上背书。**

## 二、环境变量：Vite 的规则和别处不一样

很多从 webpack/CRA 来的同学在这里栽跟头。Vite 的环境变量有三条硬规则。

### 规则 1：只有 `VITE_` 前缀的变量会进客户端代码

这是 Vite 的默认规则，不需要额外配置；只有想把前缀改成别的值时，才需要配置 `envPrefix`。

```bash
# .env
VITE_API_BASE=https://api.example.com   # ✅ 会被注入到浏览器代码
DB_PASSWORD=super-secret                  # ❌ 不带前缀，不会进客户端，安全
```

在代码里通过 `import.meta.env` 读取：

```js
console.log(import.meta.env.VITE_API_BASE); // https://api.example.com
console.log(import.meta.env.DB_PASSWORD);   // undefined —— 设计如此
```

注意，这条规则只限制**变量是否注入浏览器代码**，不代表服务端不能读取。Vite 内部已经具备类似 `dotenv` 的加载能力，不需要你额外安装 `dotenv`；它会按模式加载 `.env*` 文件，服务端侧仍然可以通过 `process.env` 读取非 `VITE_` 变量：

```js
// server.js
console.log(process.env.DB_PASSWORD);
```

所以，密钥可以放在服务端环境变量里使用，但不要加 `VITE_` 前缀暴露给客户端。

**这是 Vite 防泄露的安全闸门**：客户端代码最终会下发到用户浏览器，任何注入进去的值都是公开的。Vite 默认只暴露 `VITE_` 前缀，逼你显式选择“哪些变量可以公开”。前缀可用 `envPrefix` 改，但**不要把它改空或改成会包含密钥的前缀**。

> 真实事故：有人为了图省事把 `envPrefix` 设成 `''`，结果整份 `.env`（含数据库密码、第三方 token）全被打进了 JS bundle 上线。务必只把需要公开的变量加 `VITE_` 前缀，密钥类变量绝不进客户端代码。

### 规则 2：内置变量

无需定义即可用：

| 变量 | 含义 |
|---|---|
| `import.meta.env.MODE` | 当前模式（`development` / `production` / 自定义） |
| `import.meta.env.DEV` | 是否开发环境（boolean） |
| `import.meta.env.PROD` | 是否生产环境（boolean） |
| `import.meta.env.BASE_URL` | 配置里的 `base` |
| `import.meta.env.SSR` | 是否运行在服务端（见 [06 SSR](./06-SSR库模式多页应用.md)） |

`import.meta.env.PROD` 在 build 时会被**静态替换为字面量**，配合 tree-shaking 能让 `if (import.meta.env.DEV) { ...调试代码... }` 在生产产物里被整段删掉——这是“零成本调试日志”的常用技巧。

### 规则 3：`.env` 文件的加载优先级与模式挂钩

Vite 按下面的顺序加载 `.env` 文件（后者覆盖前者），且 `.env.[mode]` 与当前 mode 绑定：

```
.env                # 所有情况都加载
.env.local          # 所有情况都加载，但被 git 忽略（放本地私密配置）
.env.[mode]         # 仅该 mode 加载，如 .env.production
.env.[mode].local   # 仅该 mode 加载且被 git 忽略
```

**约定**：`*.local` 加进 `.gitignore`，用于放不该提交的本地值。`.env.production`、`.env.development` 提交进仓库放公共配置。

## 三、多模式（mode）：不止 development / production

`mode` 是 Vite 区分环境的核心维度，但它**不等于** `NODE_ENV`，也**不限于**两种。

### 默认行为

- `vite` / `vite dev` → mode 默认 `development`
- `vite build` → mode 默认 `production`

### 自定义模式：解决“多套环境”的真实需求

现实里你常有 `development` / `staging`（预发） / `production` 三套甚至更多环境，它们 API 地址、埋点开关都不同。用 `--mode` 指定：

```bash
vite build --mode staging
```

这会让 Vite 额外加载 `.env.staging`，并把 `import.meta.env.MODE` 设为 `staging`。于是你可以：

```
.env.staging
VITE_API_BASE=https://staging-api.example.com
VITE_ENABLE_DEBUG=true
```

配合 `package.json` 脚本固化：

```json
{
  "scripts": {
    "build": "vite build",
    "build:staging": "vite build --mode staging"
  }
}
```

### 在配置里按命令/模式分支

`defineConfig` 支持函数形式，拿到 `command` 与 `mode` 动态返回配置：

```ts
import { defineConfig, loadEnv } from 'vite';

export default defineConfig(({ command, mode }) => {
  // loadEnv 在「配置阶段」读取 .env（注意：第三个参数控制前缀过滤）
  const env = loadEnv(mode, process.cwd(), '');
  return {
    define: {
      __APP_VERSION__: JSON.stringify(env.APP_VERSION ?? 'dev'),
    },
    build: {
      sourcemap: command === 'build' && mode !== 'production',
    },
  };
});
```

> 注意区分两类代码：**Vite 处理的应用代码**里用 `import.meta.env` 读取 Vite 提供的环境变量，其中客户端代码只能拿到 `VITE_*` 这类允许公开的变量；**Node 直接执行的代码**里用 `process.env`，比如独立 Node 脚本或后端服务。Vite 会在解析配置后自动加载 `.env*` 并提供给应用代码；但如果 `vite.config.ts` 自己就要根据 `.env` 决定端口、代理、插件开关、`define` 等配置，需要在配置阶段手动调用 `loadEnv(...)`。

## 四、动手：多模式与环境变量

配套代码：[第一阶段-使用篇/01-核心使用/03-环境变量与多模式/](https://github.com/XY0987/vite_brochure_code_public/tree/main/第一阶段-使用篇/01-核心使用/03-环境变量与多模式/)

demo 准备了 `.env`、`.env.development`、`.env.production`、`.env.staging`，页面上实时显示当前 `MODE`、各环境变量值，并故意放了一个**不带前缀的“密钥”**让你验证它读不到。`vite.config.ts` 还演示了在 **Node 直接执行的配置文件**里用 `loadEnv` 读取不带前缀的 `APP_VERSION`、再经 `define` 注入为编译期常量 `__APP_VERSION__`，正好对照上面「Vite 处理的应用代码 vs Node 直接执行的代码」两种读法。

```bash
cd vite_brochure_code_public/第一阶段-使用篇/01-核心使用/03-环境变量与多模式
npm install

npm run dev                  # MODE=development，读 .env + .env.development
npm run build && npm run preview     # MODE=production
npm run build:staging && npm run preview  # MODE=staging，API 地址不同
```

观察点：
- 页面显示的 `VITE_API_BASE` 在 dev / production / staging 三种构建下不同。
- 页面尝试读取 `SECRET_TOKEN`（无前缀）显示 `undefined` —— 安全闸门生效。
- 生产构建产物里全文搜不到 `SECRET_TOKEN` 的值（验证没泄露）。

## 五、本节小结

- 三命令边界：`dev`（开发体验，非生产）、`build`（产物正确性）、`preview`（只验证产物本身，非真实生产环境）。
- 环境变量铁律：**只有 `VITE_` 前缀进客户端**，这是防密钥泄露的核心机制。
- `.env` 按 `mode` 分层加载，`*.local` 不提交。
- `mode` 可自定义（如 `staging`），用 `--mode` 切换；Vite 处理的应用代码用 `import.meta.env`，Node 直接执行的配置文件按需用 `loadEnv`。

## 六、可直接用于项目的 checklist

- [ ] 上线前流程固定为：`build` → `preview` 自检 → 部署到真实环境再验证（不拿 dev/preview 给线上背书）。
- [ ] 所有要暴露给前端的变量都加了 `VITE_` 前缀；密钥类变量**没有**前缀。
- [ ] `.env.*.local` 已加入 `.gitignore`；公共配置 `.env.production` 等已提交。
- [ ] 多环境用 `--mode` + `.env.[mode]` 管理，并在 `package.json` 脚本里固化。
- [ ] Vite 处理的应用代码里读变量用 `import.meta.env`，Node 直接执行的配置文件里按需用 `loadEnv`，没有混用。
- [ ] 构建后全局搜索过敏感值，确认没有泄露进产物。
