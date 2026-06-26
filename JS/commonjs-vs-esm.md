---
type: review
---
# 模块化（CommonJS 与 ES Module）

## 1. 核心概念：为什么有两种模块系统？

- **CommonJS（CJS）**：Node.js 诞生时 JavaScript 没有语言级模块规范，Node 自己设计了一套。用 `require()` / `module.exports`，运行时动态加载。
- **ES Module（ESM）**：ES6（2015）将模块作为语言标准写入规范。用 `import` / `export`，编译时静态解析。

两者的**本质差异**在于一个关键词：**静态 vs 动态**。
- ESM 的 `import`/`export` 是**语法结构**，引擎在执行代码前就能确定依赖关系 → 可以做 tree-shaking、静态分析。
- CJS 的 `require()` 是**普通函数调用**，只有执行到那一行才知道要加载什么模块 → 天然支持动态加载，但无法静态分析。

---

## 2. 对比表

| 对比维度 | CommonJS | ES Module |
|---|---|---|
| **语法** | `require()` / `module.exports` | `import` / `export` |
| **依赖解析时机** | **运行时**（执行到 `require()` 才解析） | **编译时**（执行前就确定依赖图） |
| **模块导出机制** | `module.exports` 是一个对象，`require()` 返回该对象的**引用**（解构时对原始类型按值复制） | **活绑定**（import 变量是 export 变量的只读引用，始终指向最新值） |
| **this 指向** | 当前模块 `module` 对象 | `undefined`（严格模式） |
| **严格模式** | 默认非严格模式 | 默认严格模式 |
| **循环依赖** | 已执行的部分返回（可能拿到不完整的 exports） | 因为是活绑定，求值完成后能拿到最终值 |
| **动态导入** | 天然支持（`require()` 是函数，可放在任意位置） | 需要 `import()` 函数（返回 Promise） |
| **Tree-shaking** | ❌ 无法静态分析依赖，不能 tree-shake | ✅ 静态结构，打包工具可分析出未使用的导出并剔除 |
| **顶层 await** | ❌ 不支持 | ✅ 支持 |
| **环境** | Node.js 原生支持 | 浏览器原生支持；Node.js 需 `.mjs` 或 `package.json` 中 `"type": "module"` |

---

## 3. 模块加载机制详解

### CommonJS — 运行时加载 + 返回 exports 对象引用

```js
// counter.js
let count = 0;
module.exports = { count, increment: () => count++ };

// main.js
const { count, increment } = require('./counter');
// 等价于：const temp = require('./counter'); const count = temp.count; const increment = temp.increment;
increment();
console.log(count); // 0 ❌ 不是 1！
// count 是原始类型，在解构时已经按值复制为 0，与模块内部的 count 无关
// 但 increment 是函数，它闭包引用的是模块内部的 count 变量，所以 increment() 确实让模块内部的 count 变成了 1
```

**运行时加载流程：**
1. 遇到 `require('./counter')` → 同步读取并执行 `counter.js` 的代码
2. `counter.js` 执行完毕 → `module.exports` 被确定为 `{ count: 0, increment: fn }`
3. **缓存**：模块结果被缓存到 `require.cache`，后续 `require('./counter')` 直接返回缓存
4. 返回 `module.exports` 的**引用**（不是深拷贝，是同一个对象）
5. `main.js` 中解构时，`count = 0`（原始类型按值复制），`increment = fn`（函数按引用复制）

关键点：`require()` 是**普通函数调用**，可以出现在 `if`、`for`、函数体等任意位置。

### ES Module — 编译时解析 + 活绑定

```js
// counter.js
export let count = 0;
export function increment() { count++; }

// main.js
import { count, increment } from './counter.js';
increment();
console.log(count); // 1 ✅ 活绑定，拿到最新值
```

**编译时解析流程（三个阶段，代码执行前完成前两步）：**

1. **解析（Parse）**：静态扫描所有 `import` / `export` 语句，构建**模块依赖图**（不执行任何业务逻辑）
2. **实例化（Instantiate）**：在内存中为每个模块创建**模块环境记录**（Module Environment Record），将 import 变量和 export 变量绑定到**同一块内存地址**（活绑定的物理基础）
3. **求值（Evaluate）**：按依赖顺序执行模块代码，填充变量的实际值

```
模块环境记录中的绑定关系：

counter.js 的模块环境记录         main.js 的模块环境记录
┌──────────────────────┐        ┌──────────────────────┐
│ export count ──────────┐      │ import count ─────────┐│
│                        ├──────┼────────────────────────┘│
│ export increment ──────┐      │ import increment ──────┐│
│                        ├──────┼────────────────────────┘│
└──────────────────────┘        └──────────────────────┘
        同一块内存 ← 活绑定
```

关键点：`import` 必须在**模块顶层**，不能放在 `if` 或函数内，因为编译阶段就要确定依赖图（`import()` 是唯一例外，返回 Promise）。

---

## 4. 为什么 ESM 能 Tree-shake 而 CJS 不能？

```js
// utils.js — CommonJS
const a = () => 'a';
const b = () => 'b';
module.exports = { a, b };

// main.js
const { a } = require('./utils'); // 打包工具无法确定 b 是否有副作用，不敢删除
```

```js
// utils.js — ES Module
export const a = () => 'a';
export const b = () => 'b';

// main.js
import { a } from './utils.js'; // 打包工具静态分析到 b 未被导入，可以安全删除
```

ESM 的 `export` 是声明式的语法结构，打包工具（Webpack/Rollup/esbuild）在**不执行代码**的情况下就能分析出哪些导出被使用了。CJS 的 `module.exports` 是运行时赋值的表达式，静态分析工具无法确定导出内容，因此不能安全地 tree-shake。

---

## 5. 总结

```
CommonJS:  require() → 运行时执行模块 → 返回 exports 对象引用 → 解构时原始类型按值复制
ES Module: import  → 编译时建立活绑定 → import 变量始终指向 export 变量 → 始终拿到最新值
```

**一句话：CJS 是「运行时的函数调用」，ESM 是「编译时的语法结构」。**
