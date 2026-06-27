---
type: review
---
# Vue 2 的事件机制（$on / $emit / $off / $once）

## 一、核心概念

### 1. Vue 2 事件系统的定位

Vue 2 的事件系统有两层含义：

| 层级 | 说明 | API |
|------|------|-----|
| **DOM 事件** | 模板上的 `@click`、`@input` 等，绑定的是浏览器原生事件 | `v-on` 指令 |
| **组件自定义事件** | 父子组件之间的通信机制，`$on` / `$emit` / `$off` / `$once` | 实例方法 |

这里的 `$on` / `$emit` / `$off` / `$once` 指的是**组件自定义事件**，不是 DOM 事件。

### 2. 事件总线（Event Bus）模式

`$on` / `$emit` 最常见的使用场景是**事件总线**：

```js
// 创建一个独立的 Vue 实例作为事件总线
const bus = new Vue()

// 组件 A：监听事件
bus.$on('update', (data) => { console.log(data) })

// 组件 B：触发事件
bus.$emit('update', { id: 1 })
```

> Vue 3 移除了实例事件方法后，事件总线模式不再被官方推荐。

---

## 二、`$on` 和 `$emit` 的实现原理

### 1. 数据结构：`vm._events`

每个 Vue 实例在初始化时会创建一个 `_events` 对象，用于存储所有注册的事件回调：

```js
// 源码位置：src/core/instance/events.js
Vue.prototype.$on = function (event, fn) {
  const vm = this
  // _events 是一个对象，key 是事件名，value 是回调数组
  // 用数组是因为同一个事件可以注册多个回调
  if (Array.isArray(event)) {
    // 支持批量注册：$on(['click', 'focus'], handler)
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
  }
  return vm
}
```

**`_events` 的数据结构：**

```js
vm._events = {
  'update': [fn1, fn2],       // 同一事件可以有多个回调
  'delete': [fn3],
  'click':  [fn4]
}
```

**关键点：**
- `_events` 是一个普通对象，以事件名为 key，回调数组为 value
- 同一事件可以注册多个回调，按注册顺序 push 到数组中
- `$on` 返回 `vm` 本身，支持链式调用

### 2. `$emit` 的触发逻辑

```js
// 源码位置：src/core/instance/events.js
Vue.prototype.$emit = function (event) {
  const vm = this
  let cbs = vm._events[event]

  if (cbs) {
    // 将类数组 arguments 转为真数组，去掉第一个参数（事件名）
    cbs = cbs.length > 1 ? toArray(arguments, 1) : [arguments[1]]
    // 依次调用所有回调，this 指向 vm
    for (let i = 0, l = cbs.length; i < l; i++) {
      try {
        cbs[i].apply(vm, cbs)
      } catch (e) {
        handleError(e, vm, `event handler for "${event}"`)
      }
    }
  }
  return vm
}
```

**触发流程：**

```
$emit('update', data)
    │
    ▼
从 vm._events['update'] 取出回调数组 cbs
    │
    ▼
将 arguments 转为数组（去掉事件名，只保留参数）
    │
    ▼
遍历 cbs，逐个 cbs[i].apply(vm, args) 调用
    │
    ▼
返回 vm（支持链式调用）
```

**关键点：**
- `$emit` 通过 `apply` 调用回调，`this` 指向组件实例
- 回调按注册顺序依次同步执行
- 回调中的异常会被 `try/catch` 捕获并通过 `handleError` 处理，不会中断后续回调
- 如果没有注册该事件的回调，`$emit` 不会报错，静默返回

---

## 三、`$off` 和 `$once` 的实现

### 1. `$off`：移除事件监听

```js
Vue.prototype.$off = function (event, fn) {
  const vm = this

  // 无参数：清除所有事件
  if (!arguments.length) {
    vm._events = Object.create(null)
    return vm
  }

  // 支持批量取消
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$off(event[i], fn)
    }
    return vm
  }

  const cbs = vm._events[event]
  if (!cbs) return vm  // 该事件没有注册过，直接返回

  // 只传事件名，不传 fn：移除该事件的所有回调
  if (!fn) {
    vm._events[event] = null
    return vm
  }

  // 传了 fn：只移除特定回调
  if (fn) {
    let cb
    let i = cbs.length
    while (i--) {
      cb = cbs[i]
      // 支持 $once 注册的 wrapped fn 的匹配
      if (cb === fn || cb._fn === fn) {
        cbs.splice(i, 1)
        break
      }
    }
  }
  return vm
}
```

**三种用法：**

| 调用 | 效果 |
|------|------|
| `$off()` | 清除所有事件 |
| `$off('update')` | 移除 `update` 的所有回调 |
| `$off('update', fn)` | 只移除 `update` 中的 `fn` 回调 |

### 2. `$once`：一次性监听

```js
Vue.prototype.$once = function (event, fn) {
  const vm = this

  function on () {
    vm.$off(event, on)   // 先取消自身
    fn.apply(vm, arguments)  // 再执行原始回调
  }

  // 将原始 fn 挂在 on._fn 上，供 $off 匹配使用
  on._fn = fn

  vm.$on(event, on)
  return vm
}
```

**核心思路：** `$once` 不是注册原始 `fn`，而是注册一个包装函数 `on`。`on` 被触发时会先 `$off` 自己，再执行 `fn`，实现"只触发一次"的效果。

```
$once('update', fn)
    │
    ▼
创建包装函数 on（on 内部先 $off 再调 fn）
    │
    ▼
$on('update', on)  ← 实际注册的是 on
    │
    ▼
$emit('update') 时触发 on
    │
    ▼
on 执行 → $off('update', on) → fn(args)
    │
    ▼
下次 $emit('update') 时，on 已被移除，fn 不再触发
```

**为什么需要 `on._fn = fn`？**

当用户手动调用 `$off('update', fn)` 取消 `$once` 注册的事件时，`_events` 里存的是 `on` 而不是 `fn`。`$off` 通过 `cb._fn === fn` 来匹配，确保能正确移除。

---

## 四、Vue 3 为什么移除了 `$on` / `$off` / `$once`？

### 1. 移除原因

| 原因 | 说明 |
|------|------|
| **与 Composition API 的设计哲学冲突** | Composition API 强调显式的数据流和可追踪性，隐式的事件系统不利于代码可读性 |
| **Tree-shaking 不友好** | 实例方法挂载在原型上，无法被静态分析，编译器无法判断是否被使用 |
| **事件总线是反模式** | 跨组件隐式通信导致数据流难以追踪，大型项目中容易造成"事件地狱" |
| **类型推断困难** | 动态事件名 + 回调函数，TypeScript 无法提供良好的类型提示 |
| **`$emit` 保留但语义收窄** | Vue 3 中 `$emit` 仅用于子组件向父组件触发自定义事件（配合 `emits` 选项声明），不再是通用事件系统 |

### 2. 推荐替代方案

| 场景 | Vue 2 做法 | Vue 3 推荐 |
|------|-----------|-----------|
| 父子通信 | `$emit` | `$emit`（需在 `emits` 中声明） |
| 跨组件通信 | `Event Bus`（`$on` / `$emit`） | [[pinia 的核心概念和使用方式\|Pinia]] / [[Vuex]] 状态管理 |
| 全局事件 | `Event Bus` | 外部库：`mitt` / `tiny-emitter` |
| 组件内监听 | `$once` | `onUnmounted` 钩子中手动清理，或用 `watchEffect` 的 `onCleanup` |

**用 mitt 替代事件总线：**

```js
import mitt from 'mitt'
const emitter = mitt()

// 监听
emitter.on('update', (data) => { console.log(data) })

// 触发
emitter.emit('update', { id: 1 })

// 取消
emitter.off('update', handler)
```

`mitt` 是一个不到 200 字节的事件库，API 简洁，且是独立模块，天然支持 Tree-shaking。

---

## 五、总结

### 问题①：`$on` 和 `$emit` 的实现原理

- `$on` 将回调函数 push 到 `vm._events[eventName]` 数组中
- `$emit` 从 `vm._events[eventName]` 取出回调数组，用 `apply` 依次同步调用
- `$off` 通过 `splice` 移除数组中的特定回调，或直接置 `null` 清空
- `$once` 注册一个包装函数，触发时先 `$off` 自己再执行原始回调

### 问题②：Vue 3 移除原因及替代方案

- 移除核心原因：事件总线是反模式 + Tree-shaking 不友好 + 与 Composition API 冲突
- 父子通信仍用 `$emit`（需声明 `emits`）
- 跨组件通信用 Pinia 或外部事件库（`mitt`）
