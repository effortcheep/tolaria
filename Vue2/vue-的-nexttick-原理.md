---
type: review
---

# Vue 的 $nextTick 原理

两个小问：\
**① $nextTick 是做什么的？什么场景下需要用到它？**\
**② $nextTick 底层是怎么实现的？为什么它能把回调放到"DOM更新之后"执行？**

---

## 一、前置概念：Vue 的异步更新队列

### 1.1 为什么需要异步更新？

Vue **不是**数据一变就立刻操作 DOM。如果每次数据变化都同步更新 DOM，会导致：

```js
this.name = '张三'  // → 触发 DOM 更新（第1次重渲染）
this.age = 18       // → 触发 DOM 更新（第2次重渲染）
this.city = '北京'  // → 触发 DOM 更新（第3次重渲染）
// 三次数据变化 → 三次 DOM 更新 → 性能浪费！
```

Vue 的做法：**将数据变化收集起来，合并为一次 DOM 更新**。

### 1.2 异步更新队列（Queue）

Vue 2 的更新机制：

```
数据变化
  ↓
Watcher.getter() 触发 setter
  ↓
dep.notify() 通知所有 Watcher
  ↓
Watcher 入队（queueWatcher）
  ↓
同一个 tick 内的多次数据变化，Watcher 只入队一次
  ↓
当前同步代码执行完毕
  ↓
nextTick(flushSchedulerQueue) 批量更新
  ↓
DOM 更新完成
```

**关键点：queueWatcher 不是立即执行更新，而是把 Watcher 推入一个队列，等当前同步代码全部执行完后，再统一 flush 队列执行更新。**

### 1.3 为什么要用 $nextTick？

由于 DOM 更新是异步的，在数据变化后**立刻**去读取 DOM，拿到的是**旧值**：

```js
this.message = '新内容'

// ❌ 此时 DOM 还没更新，读到的是旧内容
console.log(this.$el.textContent)  // '旧内容'

// ✅ 在 nextTick 回调中，DOM 已经更新完毕
this.$nextTick(() => {
  console.log(this.$el.textContent)  // '新内容'
})
```

---

## 二、$nextTick 的使用场景

### 2.1 典型场景一：数据变化后操作 DOM

```js
addMessage(newMsg) {
  this.messages.push(newMsg)
  // ❌ 直接操作：新 DOM 节点还没渲染出来
  this.$refs.list.scrollTop = this.$refs.list.scrollHeight

  // ✅ 用 nextTick：等 DOM 更新后再滚动
  this.$nextTick(() => {
    this.$refs.list.scrollTop = this.$refs.list.scrollHeight
  })
}
```

### 2.2 典型场景二：获取更新后的 DOM 尺寸/状态

```js
showDetail() {
  this.visible = true
  // ❌ 元素还没渲染出来，宽度为 0
  console.log(this.$refs.detail.offsetWidth)

  // ✅ 等待渲染完成
  this.$nextTick(() => {
    console.log(this.$refs.detail.offsetWidth)  // 正确值
  })
}
```

### 2.3 典型场景三：在 created 中操作 DOM

```js
export default {
  created() {
    // ❌ created 阶段 DOM 还未挂载
    // this.$refs.xxx 是 undefined

    // ✅ 用 nextTick 延迟到 DOM 挂载后
    this.$nextTick(() => {
      console.log(this.$refs.xxx)  // 可以访问
    })
  }
}
```

### 2.4 典型场景四：第三方库初始化

```js
mounted() {
  this.visible = true
  this.$nextTick(() => {
    echarts.init(this.$refs.chart)
  })
}
```

---

## 三、$nextTick 底层原理

### 3.1 核心思路

$nextTick(cb) 的本质：**把回调函数放到微任务队列中，等当前宏任务（包括同步代码和 DOM 更新）执行完毕后，再执行回调。**

```
┌─────────────────────────────────────────┐
│           同步代码执行                    │
│  this.message = '新内容'                 │
│  this.$nextTick(cb)  ← 回调入队          │
├─────────────────────────────────────────┤
│         微任务队列                        │
│  flushSchedulerQueue（DOM 更新）         │
│  cb（用户回调）  ← 紧随其后执行           │
├─────────────────────────────────────────┤
│         宏任务队列                        │
│  setTimeout / setInterval ...           │
└─────────────────────────────────────────┘
```

### 3.2 降级策略（优雅降级）

Vue 2 依次尝试使用以下 API 创建微任务：

| 优先级 | API | 说明 |
|:---:|------|------|
| 1 | Promise | 原生微任务，现代浏览器首选 |
| 2 | MutationObserver | 微任务，兼容性好 |
| 3 | setImmediate | 宏任务，IE 专属，比 setTimeout 快 |
| 4 | setTimeout(fn, 0) | 兜底方案，宏任务 |

**为什么优先微任务？**
- 微任务在当前宏任务结束后、下一个宏任务开始前执行
- 能在**同一轮事件循环**中拿到更新后的 DOM
- setTimeout 是宏任务，可能会被其他宏任务插队

### 3.3 源码精简版

```js
const callbacks = []
let pending = false

function nextTick(cb) {
  callbacks.push(cb)
  if (!pending) {
    pending = true
    timerFunc()
  }
}

function flushCallbacks() {
  pending = false
  const copies = callbacks.slice()
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc
if (typeof Promise !== 'undefined') {
  const p = Promise.resolve()
  timerFunc = () => { p.then(flushCallbacks) }
} else if (typeof MutationObserver !== 'undefined') {
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, { characterData: true })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
} else if (typeof setImmediate !== 'undefined') {
  timerFunc = () => { setImmediate(flushCallbacks) }
} else {
  timerFunc = () => { setTimeout(flushCallbacks, 0) }
}
```

### 3.4 $nextTick 在 Vue 实例上的挂载

```js
Vue.prototype.$nextTick = function (cb) {
  return nextTick(cb, this)
}
```

返回值是一个 **Promise**，所以也支持 async/await 写法：

```js
// 回调写法
this.$nextTick(() => { /* DOM 已更新 */ })

// Promise 写法
this.$nextTick().then(() => { /* DOM 已更新 */ })

// async/await 写法
async handleClick() {
  this.message = '新内容'
  await this.$nextTick()
  console.log(this.$el.textContent)  // '新内容'
}
```

---

## 四、$nextTick 与 DOM 更新的关系

### 4.1 完整时序图

```
用户代码                  Vue 内部                     浏览器
─────────                ────────                    ───────
this.msg = 'B'
                         setter 触发
                         watcher.update()
                         queueWatcher(watcher)
                         watcher 入队
                         pending = false?
                           → 是，调用 timerFunc()
                           → callbacks.push(flush)
this.$nextTick(cb)         → callbacks.push(cb)
                           → pending 已为 true，不重复安排
───────── 同步代码结束 ─────────────────────────────
                         微任务执行：
                           flushCallbacks()
                           ↓
                           flush()（DOM 更新）
                           cb()（用户回调）
───────────────────────────────────────────────────
                           ↓
                         浏览器渲染新 DOM（paint）
```

### 4.2 关键理解

1. **flushSchedulerQueue（DOM 更新）和用户 $nextTick 的回调都在同一个微任务队列中**
2. Vue 内部先 push(flush)，用户再 push(cb)，所以 flush 先于 cb 执行
3. 这就是为什么在 $nextTick 回调中能拿到**更新后的 DOM**

### 4.3 一个容易踩的坑

```js
this.message = 'A'
this.$nextTick(() => {
  console.log('cb1:', this.message)  // 'A'
})

this.message = 'B'
this.$nextTick(() => {
  console.log('cb2:', this.message)  // 'B'
})

// 两次 nextTick 的回调都会在 DOM 更新完成后执行
// 但 message 只有一个 Watcher，最终更新的是 'B'
// cb1 拿到的是最终状态 'B'，不是中间状态 'A'
```

**注意：Vue 2 的 Watcher 是异步批量更新的。多次赋值只会触发一次 DOM 更新（最终值），中间值不会产生独立的 DOM 快照。**

---

## 五、Vue 3 中的 $nextTick

```js
import { nextTick } from 'vue'

this.$nextTick(cb)

// Vue 3 的 nextTick 基于 Promise
export function nextTick(fn) {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

**Vue 3 的变化：**
- 不再需要降级策略（Vue 3 要求 Promise 支持，IE 已被放弃）
- 直接使用 Promise.resolve().then()
- 可以从 vue 包中直接导入 nextTick，不再只挂在实例上

---

## 六、完整回答

### Q1：$nextTick 是做什么的？什么场景下需要用到它？

**做什么：** $nextTick(cb) 将回调延迟到下一次 DOM 更新刷新之后执行。它利用微任务机制，确保回调执行时 DOM 已经反映了最新的数据变化。

**使用场景：**
1. **数据变化后操作 DOM** — 如更新列表后滚动到底部
2. **获取更新后的 DOM 状态** — 如读取元素尺寸、位置
3. **在 created 中访问 DOM/refs** — created 阶段 DOM 未挂载，需 nextTick 延迟
4. **第三方库需要就绪的 DOM** — 如 ECharts、地图等在数据驱动显示后初始化

**一句话总结：** 凡是"数据变了之后要操作 DOM"的场景，都需要 $nextTick。

---

### Q2：$nextTick 底层是怎么实现的？为什么它能把回调放到"DOM 更新之后"执行？

**底层实现：**
1. Vue 内部维护一个 callbacks 数组
2. 调用 $nextTick(cb) 时，把 cb 推入数组
3. 首次调用时，通过 timerFunc 安排一次微任务来批量执行所有回调
4. timerFunc 按优先级选择：Promise → MutationObserver → setImmediate → setTimeout

**为什么能在 DOM 更新之后执行：**

Vue 的 DOM 更新（flushSchedulerQueue）和用户的 $nextTick 回调都在**同一个微任务队列**中。Vue 内部先安排 DOM 更新入队，用户再入队回调，执行顺序保证为：

```
微任务执行：DOM 更新（flush） → 用户回调（cb）
```

由于 Vue 内部的 flushSchedulerQueue 一定先于用户的 cb 被 push 到 callbacks 中，所以当微任务触发 flushCallbacks 时，DOM 更新一定先执行，用户的 cb 后执行，从而保证了回调执行时 DOM 已经是最新状态。

**本质上就是利用了 JavaScript 事件循环的微任务机制，把"更新 DOM"和"用户回调"都放到微任务队列中，通过入队顺序保证执行顺序。**
