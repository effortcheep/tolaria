---
type: review
related_to: "[[虚拟 DOM 与 Diff 算法]]"
---

# Vue 的虚拟 DOM（Virtual DOM）

## 一、真实 DOM 与性能瓶颈

### 1.1 什么是 DOM

DOM（Document Object Model）是浏览器将 HTML 解析后生成的**树形结构**，JavaScript 通过 DOM API 来读取和修改页面内容。

```
HTML: <div id="app"><p>Hello</p></div>

      对应的 DOM 树：
           document
              |
            html
              |
            div#app
              |
             p
              |
           "Hello"
```

### 1.2 直接操作 DOM 为什么慢

**问题不在于"单次操作"，而在于"频繁操作"带来的连锁开销：**

| 开销来源 | 说明 |
|---------|------|
| **JS 引擎 ↔ 渲染引擎通信** | 每次 DOM 操作都要跨线程调用 |
| **重排（Reflow）** | 修改布局属性（宽高/位置），浏览器重新计算几何信息 |
| **重绘（Repaint）** | 修改外观属性（颜色/背景），浏览器重新绘制像素 |
| **浏览器优化被打破** | 浏览器会批量处理 DOM 操作，但 JS 的同步代码会打断这个优化 |

### 1.3 核心矛盾

> **状态频繁变化** ↔ **DOM 操作代价高**

手动优化繁琐且容易出错，需要一种机制来**自动、高效地将状态变化映射到 DOM 更新**。

---

## 二、虚拟 DOM 的本质

### 2.1 定义

> **虚拟 DOM（Virtual DOM）就是用一个普通的 JavaScript 对象来描述真实 DOM 节点的结构。**

```html
<!-- 真实 DOM -->
<div class="box" id="app">Hello</div>
```

```js
// 对应的虚拟 DOM（VNode）
{
  tag: 'div',
  data: { class: 'box', id: 'app' },
  children: undefined,
  text: 'Hello',
  elm: <真实 DOM 引用>,
  ...
}
```

### 2.2 对比理解

| | 真实 DOM | 虚拟 DOM |
|--|---------|---------|
| **本质** | 浏览器维护的 C++ 对象 | JS 普通对象 |
| **创建代价** | 重（需要渲染引擎参与） | 轻（就是声明一个对象） |
| **操作方式** | document.createElement 等 API | 直接操作 JS 对象属性 |
| **存储位置** | 渲染引擎内存 | JS 堆内存 |

---

## 三、为什么需要虚拟 DOM

### 3.1 虚拟 DOM 解决了什么问题

| 问题 | 虚拟 DOM 的解决方案 |
|------|-------------------|
| **频繁操作 DOM 性能差** | 先在 JS 层面计算差异，再一次性更新真实 DOM |
| **手动优化繁琐** | 框架自动 Diff，开发者只管声明状态 |
| **跨平台困难** | VNode 是平台无关的 JS 对象，可以渲染到不同平台（Web / Native / Canvas） |

### 3.2 虚拟 DOM 的真正价值

> **虚拟 DOM 的核心价值不是"性能"，而是"声明式编程" + "可预测的更新"。**

```js
// 命令式（手动操作 DOM）
document.getElementById('count').innerText = count
document.getElementById('btn').disabled = count >= 10

// 声明式（虚拟 DOM 自动 diff 更新）
template: `<button :disabled="count >= 10">{{ count }}</button>`
```

开发者只需要描述"UI 应该长什么样"，框架自动帮你算出哪些 DOM 需要更新。

### 3.3 虚拟 DOM 的性能定位

| 场景 | 虚拟 DOM vs 手动优化 |
|------|---------------------|
| 首次渲染 | 略慢（多了创建 VNode + Diff 的步骤） |
| 大量数据小范围更新 | 接近手动优化（Diff 找到差异，精准更新） |
| 少量数据大范围更新 | 可能更慢（Diff 开销 > 直接替换） |
| 极端优化场景 | 手动操作更快（人可以做更精准的判断） |

> **注意**：虚拟 DOM 并不是"比直接操作 DOM 快"。它的价值是：在**声明式编程**的前提下，提供一个**还不错的性能保底**。

### 3.4 跨平台能力

虚拟 DOM 是纯 JS 对象，不依赖浏览器 API，可以渲染到不同平台：
- 浏览器（真实 DOM）
- 服务端（SSR → HTML 字符串）
- 原生应用（Weex/UniApp → Native 视图）
- Canvas / WebGL

---

## 四、虚拟 DOM 的工作原理

### 4.1 核心三步

```
① 创建 VNode    状态变化时，生成新的虚拟 DOM 树
     ↓
② Diff          新旧两棵虚拟 DOM 树对比，找出差异
     ↓
③ Patch          将差异应用到真实 DOM 上
```

### 4.2 详细流程

```
初始渲染：
  模板 → 编译 → render 函数 → VNode 树 → 创建真实 DOM → 挂载到页面

数据更新：
  状态变化 → 重新执行 render → 新 VNode 树
       ↓
  新旧 VNode 树 Diff（同层比较）
       ↓
  找出差异节点
       ↓
  批量更新真实 DOM（Patch）
```

### 4.3 Diff 策略：同层比较

> **只做同层比较，不做跨层移动。**

```
旧树:           新树:
  A               A
 / \             / \
B   C           B   D
                  \
                   C

同层比较：
  A === A → 相同，比较子节点
  B === B → 相同
  C !== D → 不同，替换

跨层移动（Vue 2 不会做）：
  C 移动到 D 的子节点 → Vue 2 会销毁 C，创建新的 D 和 C
```

### 4.4 Vue 2 的双端比较

```
旧子节点: [oldStart, ..., oldEnd]
新子节点: [newStart, ..., newEnd]

① oldStart vs newStart  → 头头比较
② oldEnd   vs newEnd    → 尾尾比较
③ oldStart vs newEnd    → 头尾比较
④ oldEnd   vs newStart  → 尾头比较

如果四步都没匹配 → 用 key 查找
```

### 4.5 sameVnode — 判断两个 VNode 是否"相同"

```js
function sameVnode (a, b) {
  return (
    a.key === b.key &&           // key 相同
    a.tag === b.tag &&           // 标签相同
    a.isComment === b.isComment &&  // 都是/都不是注释
    isDef(a.data) === isDef(b.data) &&  // 都有/都没有 data
    sameInputType(a, b)          // input 类型相同
  )
}
```

> 只有 `key` 和 `tag` 都相同的 VNode，才会进入 `patchVnode` 做进一步的 diff，否则直接销毁旧节点、创建新节点。

---

## 五、Vue 2 的 VNode 数据结构

### 5.1 VNode 构造函数

Vue 2 中，VNode 定义在 `src/core/vdom/vnode.js`：

```js
export default class VNode {
  constructor (
    tag,        // 标签名（如 'div'），文本节点为 undefined
    data,       // 节点数据（attrs / class / style / on 等）
    children,   // 子节点数组
    text,       // 文本内容（文本节点才有）
    elm,        // 对应的真实 DOM 节点（Patch 后赋值）
    context,    // 组件的 Vue 实例（组件节点才有）
    componentOptions, // 组件选项（props / listeners / children）
    asyncFactory // 异步组件工厂函数
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined          // 命名空间
    this.context = context
    this.fnContext = undefined    // 函数式组件上下文
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key   // 从 data 中提取 key
    this.componentOptions = componentOptions
    this.componentInstance = undefined  // 组件实例
    this.parent = undefined       // 父 VNode
    this.raw = false              // 是否包含原始 HTML
    this.isStatic = false         // 静态节点标记
    this.isRootInsert = true      // 是否作为根节点插入
    this.isComment = false        // 是否是注释节点
    this.isCloned = false         // 是否是克隆节点
    this.isOnce = false           // 是否是 v-once 节点
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }
}
```

### 5.2 常见 VNode 类型

#### 元素节点

```js
// <div id="app" class="container">Hello</div>
{
  tag: 'div',
  data: { attrs: { id: 'app' }, class: { container: true } },
  children: [{ text: 'Hello', isComment: false }],
  key: undefined
}
```

#### 文本节点

```js
// "Hello World"
{
  tag: undefined,
  text: 'Hello World',
  isComment: false,
  children: undefined
}
```

#### 注释节点

```js
// <!---->
{
  tag: undefined,
  text: 'v-if placeholder',
  isComment: true
}
```

#### 组件节点

```js
// <MyComponent :msg="hello" @click="handler" />
{
  tag: 'vue-component-1-my-component',
  data: { attrs: { msg: 'hello' }, on: { click: handler } },
  componentOptions: {
    Ctor: MyComponent,
    propsData: { msg: 'hello' },
    listeners: { click: handler },
    tag: 'my-component',
    children: undefined
  },
  componentInstance: null  // 挂载后赋值为组件实例
}
```

#### 克隆节点

```js
// 用于优化静态节点，避免重复创建
{ ...originalVNode, isCloned: true }
```

### 5.3 VNode 树结构示例

```html
<template>
  <div id="app">
    <h1>{{ title }}</h1>
    <ul>
      <li v-for="item in list" :key="item.id">{{ item.name }}</li>
    </ul>
  </div>
</template>
```

```js
// 对应的 VNode 树（简化）
{
  tag: 'div',
  data: { attrs: { id: 'app' } },
  children: [
    {
      tag: 'h1',
      children: [{ text: '标题' }]
    },
    {
      tag: 'ul',
      children: [
        { tag: 'li', key: 1, children: [{ text: '苹果' }] },
        { tag: 'li', key: 2, children: [{ text: '香蕉' }] },
        { tag: 'li', key: 3, children: [{ text: '橘子' }] }
      ]
    }
  ]
}
```

### 5.4 编译产物示例

```html
<!-- 模板 -->
<div id="app">
  <p class="title">{{ msg }}</p>
  <span>静态文本</span>
</div>
```

```js
// 编译后的 render 函数（简化）
function render() {
  return _c('div', { attrs: { id: 'app' } }, [
    _c('p', { staticClass: 'title' }, [_v(_s(msg))]),
    _c('span', [_v('静态文本')])
  ])
}

// _c = createElement → 创建元素 VNode
// _v = createTextVNode → 创建文本 VNode
// _s = toString → 转为字符串
```

### 5.5 VNode 的核心属性总结

| 属性 | 类型 | 说明 |
|------|------|------|
| `tag` | `string?` | 标签名，文本/注释节点为 `undefined` |
| `data` | `VNodeData?` | 节点数据（attrs / class / style / on / key 等） |
| `children` | `VNode[]?` | 子节点数组 |
| `text` | `string?` | 文本内容 |
| `elm` | `Element?` | 对应的真实 DOM 元素 |
| `key` | `string\|number?` | 唯一标识，用于 Diff 优化 |
| `isComment` | `boolean` | 是否是注释节点 |
| `componentOptions` | `object?` | 组件选项（组件节点才有） |
| `componentInstance` | `Vue?` | 组件实例（组件节点才有） |
| `isStatic` | `boolean` | 是否是静态节点（可跳过 Diff） |
| `isCloned` | `boolean` | 是否是克隆节点 |

---

## 六、key 的作用

```html
<!-- 没有 key：按索引比较，可能错误复用 -->
<li v-for="item in list">{{ item }}</li>

<!-- 有 key：按身份比较，精准复用 -->
<li v-for="item in list" :key="item.id">{{ item }}</li>
```

| | 无 key | 有 key |
|--|--------|--------|
| **比较方式** | 按索引逐一比较 | 按 key 精准匹配 |
| **节点复用** | 可能错误复用（状态混乱） | 精准复用（状态正确） |
| **移动检测** | 无法检测移动 | 可检测节点移动 |
| **性能** | 可能触发更多不必要的更新 | 只更新真正变化的节点 |

---

## 七、虚拟 DOM 的优缺点

### 7.1 优点

| 优点 | 说明 |
|------|------|
| **声明式编程** | 开发者只管声明状态，框架负责更新 DOM |
| **跨平台** | VNode 是 JS 对象，可渲染到 Web / Native / Canvas |
| **性能保障** | 在大多数场景下自动保证"足够好"的性能 |
| **可预测性** | 状态 → VNode → DOM，数据流清晰可追踪 |
| **组件化基础** | 组件的 render 函数返回 VNode 树，天然支持组件化 |

### 7.2 缺点

| 缺点 | 说明 |
|------|------|
| **首次渲染较慢** | 需要先创建 VNode 树，再转换为真实 DOM |
| **内存占用** | 需要同时维护新旧两棵 VNode 树 |
| **极端场景不最优** | 手动精准操作 DOM 比自动 Diff 更快 |
| **小项目过度设计** | 简单页面引入虚拟 DOM 是额外开销 |

---

## 八、回答问题

### Q1：什么是虚拟 DOM？Vue 为什么不直接操作真实 DOM，而要先操作虚拟 DOM？

**什么是虚拟 DOM：**

虚拟 DOM 就是用一个**普通的 JavaScript 对象**来描述真实 DOM 节点的结构。每个 VNode 对象包含 `tag`（标签）、`data`（属性）、`children`（子节点）、`text`（文本内容）等属性，对应真实 DOM 树的一层映射。

**为什么不直接操作真实 DOM：**

| 原因 | 说明 |
|------|------|
| **① 性能** | 频繁的 DOM 操作会触发重排重绘，VNode 操作是纯 JS 计算，代价小得多 |
| **② 自动化** | 手动判断"哪些 DOM 需要更新"非常繁琐，虚拟 DOM 的 Diff 算法自动完成 |
| **③ 声明式编程** | 开发者只需声明"状态是什么"，框架自动计算"DOM 应该怎么变" |
| **④ 跨平台** | VNode 是平台无关的 JS 对象，同一套代码可渲染到 Web、Native、Canvas |
| **⑤ 可预测性** | 状态 → VNode → DOM 的数据流清晰，避免命令式代码中难以追踪的状态 bug |

**本质理解：** 虚拟 DOM 不是为了"比手动操作更快"，而是为了在**不手动优化**的前提下，提供**足够好的性能**，同时换来**开发效率的巨大提升**。

---

### Q2：Vue 2 的虚拟 DOM 用的是什么数据结构？VNode 长什么样？

**数据结构：** Vue 2 的虚拟 DOM 采用**单链表树结构**，每个节点是一个 `VNode` 类的实例。

**VNode 的核心属性：**

```js
class VNode {
  tag,                 // 标签名（如 'div'），文本节点为 undefined
  data,                // 节点数据（attrs / class / style / on / key 等）
  children,            // 子节点数组
  text,                // 文本内容（文本节点才有）
  elm,                 // 对应的真实 DOM 节点（Patch 后赋值）
  key,                 // 唯一标识，从 data.key 提取，用于 Diff 优化
  isComment,           // 是否是注释节点
  componentOptions,    // 组件选项（组件节点才有）
  componentInstance,   // 组件实例（组件节点才有）
  isStatic,            // 是否是静态节点（可跳过 Diff）
  // ...
}
```

**VNode 的几种类型：**

| 类型 | 特征 | 示例 |
|------|------|------|
| **元素节点** | 有 `tag` 和 `children` | `{ tag: 'div', children: [...] }` |
| **文本节点** | 无 `tag`，有 `text` | `{ text: 'Hello', isComment: false }` |
| **注释节点** | 无 `tag`，`isComment: true` | `{ text: '', isComment: true }` |
| **组件节点** | `tag` 以 `vue-component-` 开头 | `{ tag: 'vue-component-1-xxx', componentOptions: {...} }` |
| **克隆节点** | `isCloned: true` | 用于优化静态节点，避免重复创建 |

**VNode 树示例：**

```html
<div id="app">
  <p>{{ msg }}</p>
</div>
```

```js
// 对应的 VNode 树（简化）
{
  tag: 'div',
  data: { attrs: { id: 'app' } },
  children: [
    {
      tag: 'p',
      children: [
        { text: 'Hello Vue', isComment: false }
      ]
    }
  ]
}
```

**关键点：** VNode 是一个轻量的 JS 对象，创建和操作的代价远小于真实 DOM。Vue 通过 `render` 函数生成 VNode 树，数据变化时生成新的 VNode 树，再通过 Diff 算法找出差异，最后一次性更新真实 DOM。

---

## 九、总结

| 问题 | 答案 |
|------|------|
| 虚拟 DOM 是什么 | 用 JS 对象描述 DOM 结构的轻量映射 |
| 为什么不直接操作 DOM | 批量更新、声明式编程、跨平台、性能保底 |
| VNode 长什么样 | 一个包含 tag/data/children/text/elm/key 等字段的 JS 对象 |
| VNode 有哪些类型 | 元素、文本、注释、组件、克隆 |
| 怎么判断两个 VNode 相同 | key === key && tag === tag（sameVnode） |
| Diff 策略 | 同层比较，不做跨层移动；Vue 2 用双端比较 |
| key 的作用 | 按身份精准匹配节点，避免错误复用 |
