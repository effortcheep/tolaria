---
type: review
---
# Webpack 与 Vite

## 1. Webpack 的核心概念

| 概念 | 说明 |
|---|---|
| **Entry** | 入口起点，Webpack 从这里开始构建依赖图（默认 `src/index.js`） |
| **Output** | 输出配置，指定打包后的文件名和输出路径（`filename`、`path`） |
| **Loader** | 模块转换器，让 Webpack 能处理非 JS 文件（如 `babel-loader` 转 ES6、`css-loader` 处理 CSS、`file-loader` 处理图片） |
| **Plugin** | 插件，在构建生命周期的钩子中执行更广泛的任务（如 `HtmlWebpackPlugin` 生成 HTML、`MiniCssExtractPlugin` 提取 CSS） |
| **Mode** | 构建模式：`development`（不压缩、有 source map）/ `production`（压缩、Tree Shaking、优化） |
| **Module / Chunk** | 模块是源码中的每个文件；Chunk 是 Webpack 打包过程中的一组模块（入口 chunk、异步 chunk、公共 chunk） |
| **Bundle** | 最终输出的文件，一个或多个 chunk 打包后的产物 |
| **DevServer** | 开发服务器，提供 HMR 热更新、代理等功能 |

**Loader vs Plugin 核心区别**：
- **Loader**：专注于**文件转换**，在模块加载阶段工作，配置在 `module.rules` 里
- **Plugin**：专注于**功能扩展**，在构建全生命周期通过钩子介入，配置在 `plugins` 里

---

## 2. Webpack vs Vite 构建速度差异

### Webpack 慢在哪

Webpack 在开发时需要**先打包再启动**：
1. 从 Entry 出发，**递归解析所有模块**的 import/require
2. 将所有模块打包成一个或多个 Bundle
3. 启动 DevServer
4. 文件改动 → **重新打包受影响的 chunk** → HMR

项目越大，模块越多，启动和热更新越慢。

### Vite 快在哪

Vite 利用浏览器原生 ESM，**跳过打包环节**：

| 阶段 | Webpack | Vite |
|---|---|---|
| 启动 | 递归构建完整依赖图 → 打包 → 启动 | 启动 DevServer → 按需编译（**几乎秒开**） |
| 首屏加载 | 一个大 Bundle | 原生 ESM 按需请求每个模块 |
| 热更新 | 重新打包受影响 chunk | 只更新变动的模块（精确 HMR） |
| 生产构建 | Webpack 自己打包 | 底层用 **Rollup** 打包（输出更优） |

**Vite 快的核心原因**：
- **开发阶段不打包**：利用浏览器原生 `<script type="module">` 直接加载 ESM 模块
- **按需编译**：浏览器请求哪个模块，Vite 才编译哪个（用 esbuild 做依赖预构建）
- **esbuild 预构建**：用 Go 写的 esbuild 预构建 `node_modules` 依赖，比 JS 打包器快 10-100 倍

---

## 3. Tree Shaking

### 是什么

Tree Shaking 是一种**死代码消除**技术：打包时移除模块中**没有被使用**的导出（export），让最终产物更小。

### 工作原理

```
源码：export function a() { ... }
      export function b() { ... }

入口：import { a } from './utils'   ← 只用了 a

产物：只包含 a 的代码，b 被 "摇掉"（shaken off）
```

**关键前提条件**：
1. **必须使用 ESM**（`import/export`）—— CommonJS（`require/module.exports`）无法静态分析，不能 Tree Shaking
2. **Webpack 的 `mode: 'production'`** 会自动开启
3. **不能有副作用的代码**：如果模块顶层有副作用（如修改全局变量），需要用 `package.json` 的 `"sideEffects": false` 声明

### Webpack 中的 Tree Shaking 流程

1. **标记阶段**：Webpack 分析模块的 export 和 import，标记哪些 export 被使用（`/*#__PURE__*/` 注解辅助）
2. **删除阶段**：Terser 等压缩工具删除未被标记的 export 代码

### Vite / Rollup 的 Tree Shaking

Rollup 的 Tree Shaking 更激进，它在模块级别做静态分析，输出更干净。Vite 生产构建用 Rollup，天然享有这个优势。
