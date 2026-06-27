---
type: review
related_to: "[[虚拟 DOM 与 Diff 算法]]"
---

# Vue 2 的模板编译过程

## 一、核心概念

### 1. 模板编译的本质

Vue 的模板（template）本质上是**一种声明式的 DSL（领域特定语言）**，浏览器并不认识 `<template>`、`v-if`、`{{ }}` 这些语法。模板编译就是把这套 DSL **翻译成**浏览器可以执行的 JavaScript 代码——具体来说，是一个返回 VNode 的 `render` 函数。

### 2. 编译的三个阶段

Vue 2 的模板编译经历三个阶段：**parse → optimize → generate**。

```
模板字符串 ──parse──▶ AST ──optimize──▶ AST（标记静态节点）──generate──▶ render 函数
```

### 3. AST（抽象语法树）

AST 是模板的树形结构表示，每个节点包含：
- `tag`：标签名
- `type`：节点类型（1=元素，2=带表达式的文本，3=纯文本）
- `children`：子节点数组
- `attrs`：属性列表
- `expression`：表达式字符串（如 `msg`）
- `text`：原始文本（如 `{{ msg }}`）
- `static`：是否静态（optimize 阶段标记）
- `parent`：父节点引用

---

## 二、第一阶段：Parse（解析）

### 目的
把模板字符串解析成 AST。

### 核心思路
用**两个指针**（`html` 字符串的 `index`）维护一个**解析栈**：

```
栈：[div]          匹配 <div>        压栈
栈：[div, p]       匹配 <p>          压栈
栈：[div, p]       遇到 </p>         弹栈，p 成为 div 的子节点
栈：[div]          遇到 </div>       弹栈，成为根节点
```

### 解析内容
| 模板语法 | AST 中的表示 |
|---|---|
| `<div id="app">` | 元素节点，`attrs: [{name:'id', value:'app'}]` |
| `{{ msg }}` | 表达式文本节点，`type: 2`，`expression: 'msg'` |
| `v-if="show"` | `if` 条件属性 |
| `v-for="item in list"` | `for` 循环属性、`alias`、`iterator` |
| `@click="fn"` | 转换为 `on: { click: 'fn' }` |
| `:class="obj"` | 转换为 `bind: { class: 'obj' }` |
| 纯文本 | 纯文本节点，`type: 3` |

### 关键函数
- `parseHTML(template, options)`：驱动整个解析过程
- `parse(template, options)`：parseHTML 的上层封装，返回 AST 根节点

---

## 三、第二阶段：Optimize（优化）

### 目的
遍历 AST，**标记静态节点**（`static: true`），为后续 diff 做准备。

### 什么是静态节点
满足以下**全部**条件的节点：
1. 不包含表达式（没有 `{{ }}`）
2. 没有 `v-if`、`v-for` 等动态指令
3. 所有属性都是静态的（没有 `:class`、`v-bind`）
4. 所有子节点也都是静态的（递归判断）

### 标记两层
- `node.static = true`：当前节点是静态的
- `node.staticRoot = true`：整个子树都是静态的（可以整个跳过 diff）

### 为什么能优化
在 patch 阶段，`staticRoot` 为 `true` 的节点**直接跳过**，不参与 diff 对比，大幅减少比较次数。

---

## 四、第三阶段：Generate（代码生成）

### 目的
把优化后的 AST 转换成 `render` 函数的代码字符串。

### 输入输出示例

**模板：**
```html
<div id="app">
  <p>{{ msg }}</p>
</div>
```

**生成的 render 函数字符串：**
```js
`with(this){return _c('div',{attrs:{"id":"app"}},[_c('p',[_v(_s(msg))])])}`
```

### 核心辅助函数（运行时）
| 函数 | 作用 |
|---|---|
| `_c(tag, data, children)` | `createElement` → 创建 VNode（元素/组件） |
| `_v(text)` | `createTextVNode` → 创建文本 VNode |
| `_s(value)` | `toString` → 把值转成字符串 |
| `_e()` | `createEmptyVNode` → 创建空（注释）VNode |

### 代码生成的递归策略
```
genElement(node)
  ├─ genNode(node)  // 根据 type 分发
  │   ├─ type 1 → genElement  // 元素
  │   ├─ type 2 → genExpression  // 带表达式的文本
  │   └─ type 3 → genText  // 纯文本
  └─ genChildren(node)  // 递归处理子节点
```

处理指令时会插入对应的代码生成逻辑：
- `v-if` → 三元表达式
- `v-for` → 循环调用 `_l()`（renderList）
- `v-bind` / `v-on` → 合并到 data 对象

---

## 五、编译在哪里发生？

| 场景 | 说明 |
|---|---|
| 完整版 Vue（含编译器） | 运行时编译：浏览器下载 vue.js → 遇到 template → 调用 `compileToFunctions` 编译 |
| 运行时版 Vue（脚手架默认） | **构建时编译**：webpack / vue-loader 在打包阶段就把 template 编译成 render 函数，浏览器拿到的已经是 JS |

```js
// 完整版：运行时编译
new Vue({
  template: '<div>{{ msg }}</div>'  // 浏览器中编译
})

// 脚手架：构建时编译（推荐）
new Vue({
  render(h) { return h('div', this.msg) }  // 已经是 render 函数
})
```

---

## 六、回答问题

### ① Vue 的模板编译分为哪几个阶段？从 `<div>{{ msg }}</div>` 到最终渲染到页面上，中间经历了什么？

**三个阶段：parse → optimize → generate**，之后进入运行时的 patch 流程：

```
模板字符串
  │
  ▼ parse
AST（抽象语法树）
  │
  ▼ optimize
标记了静态节点的 AST
  │
  ▼ generate
render 函数代码字符串 → new Function() 生成真正的 render 函数
  │
  ▼ 首次渲染
render 函数执行 → 产生 VNode 树
  │
  ▼ patch
VNode → 真实 DOM → 页面
```

**具体过程（以 `<div>{{ msg }}</div>` 为例）：**

1. **parse**：解析出 div 元素节点，其子节点是一个表达式文本节点（`type: 2`，`expression: 'msg'`）
2. **optimize**：该节点包含表达式，不是静态节点，不标记
3. **generate**：生成 `_c('div', [_v(_s(msg))])`
4. **运行时**：`with(this)` 使 `msg` 访问到 `vm.msg`，`_s(msg)` 转为字符串，`_v()` 创建文本 VNode，`_c()` 创建 div VNode
5. **patch**：遍历 VNode 树，调用 `document.createElement('div')`、`document.createTextNode(msg)`，挂载到 DOM

---

### ② 模板编译的最终产物是什么？为什么 Vue 不直接用模板字符串渲染，而要先编译？

**最终产物：一个 `render` 函数**，签名为 `function(h) { return h(tag, data, children) }`，返回一棵 VNode 树。

**为什么必须先编译，而不直接用模板字符串渲染？**

| 理由 | 说明 |
|---|---|
| **性能** | 字符串解析 + 正则匹配 + DOM 操作的开销远大于 JS 对象创建 + diff。编译一次，多次高效执行 render |
| **声明式 → 命令式** | 模板是声明式描述，render 函数是命令式代码。编译让开发者写声明式、机器执行命令式 |
| **动态能力** | `v-if`/`v-for`/`v-bind` 等指令本质上是控制流和数据绑定，必须编译成 JS 逻辑才能执行 |
| **跨平台** | render 函数生成的 VNode 是平台无关的；同一套编译产物可以对接 Web、Weex、SSR 等不同后端 |
| **编译时优化** | optimize 阶段标记静态节点，让运行时 diff 跳过不变的部分——这是纯字符串方案做不到的 |
| **与虚拟 DOM 配合** | 虚拟 DOM 的 diff 和 patch 需要结构化的 VNode，而不是字符串；编译是连接模板和虚拟 DOM 的桥梁 |

> **一句话总结**：模板编译让开发者用简洁的声明式语法写 UI，同时让框架在运行时获得结构化、可优化的 VNode 树，兼顾了开发体验和运行性能。
