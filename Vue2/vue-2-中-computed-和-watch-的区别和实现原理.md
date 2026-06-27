---
type: review
---
# Vue 2 中 computed 和 watch 的区别和实现原理

---

## 一、前置概念：Watcher 的三种模式

Vue 2 的响应式系统中，Watcher 是核心。根据 `options` 不同，Watcher 分为三种模式：

| 模式 | 触发方式 | 典型使用者 |
|------|---------|-----------|
| **computed watcher** | 惰性求值，被读取时才计算 | `computed` |
| **render watcher** | 数据变化时触发组件重新渲染 | 模板渲染 |
| **user watcher** | 数据变化时执行用户回调 | `watch` |

```js
// 源码中的关键判断（simplified）
// computed watcher
this.lazy = !!options.lazy    // 惰性求值
this.dirty = this.lazy        // 是否需要重新计算

// user watcher
this.user = !!options.user    // 用户 watcher
this.deep = !!options.deep    // 深度监听
this.immediate = !!options.immediate  // 立即执行
```

---

## 二、computed 的实现原理

### 2.1 初始化流程

```js
// 1. Vue 实例初始化时，遍历 computed 选项
function initComputed(vm, computed) {
  // 为每个 computed 属性创建一个 computed watcher
  // lazy: true → 创建时不立即求值
  for (const key in computed) {
    watchers[key] = new Watcher(vm, getter, { lazy: true })
  }
  // 2. 将 computed 属性定义到 vm 上（Object.defineProperty）
  defineComputed(vm, key, ...)
}
```

### 2.2 缓存机制核心：dirty 标志

```js
// computed watcher 的 evaluate 方法
evaluate() {
  this.value = this.get()   // 真正执行计算
  this.dirty = false        // 标记为"已缓存"
}

// 当依赖的数据变化时，触发 depend → 通知 computed watcher
update() {
  if (this.lazy) {
    this.dirty = true       // 标记为"需要重新计算"
  }
}
```

**完整缓存流程：**

```
首次读取 computed
  ↓
dirty === true（需要计算）
  ↓
执行 getter 求值，同时收集依赖
  ↓
dirty = false（已缓存）
  ↓
再次读取 computed
  ↓
dirty === false → 直接返回缓存值，不执行 getter
  ↓
依赖数据变化
  ↓
dirty = true（缓存失效）
  ↓
下次读取时重新计算
```

### 2.3 为什么 methods 没有缓存？

```js
// methods：每次调用都执行函数
this.fullName()  // 执行函数
this.fullName()  // 再次执行函数，无缓存

// computed：有 dirty 标志控制
this.fullName    // dirty=true → 计算，dirty=false
this.fullName    // dirty=false → 直接返回缓存值
```

**本质区别：**
- `methods` 是普通函数调用，每次调用都执行函数体
- `computed` 是响应式属性读取，通过 `dirty` 标志判断是否需要重新计算

---

## 三、watch 的实现原理

### 3.1 初始化流程

```js
// Vue 实例初始化时，遍历 watch 选项
function initWatch(vm, watch) {
  for (const key in watch) {
    const handler = watch[key]
    // 创建 user watcher
    vm.$watch(key, handler, options)
  }
}
```

### 3.2 核心实现

```js
// $watch 的简化实现
Vue.prototype.$watch = function (expOrFn, cb, options = {}) {
  const vm = this
  // 创建 user watcher
  const watcher = new Watcher(vm, expOrFn, {
    user: true,        // 标记为用户 watcher
    deep: options.deep,
    immediate: options.immediate,
    sync: options.sync
  })

  // immediate：立即用当前值执行一次回调
  if (options.immediate) {
    cb.call(vm, watcher.value)
  }

  // 返回一个取消观察的函数
  return function unwatchFn() {
    watcher.teardown()
  }
}
```

### 3.3 回调执行时机

```js
// user watcher 的 run 方法
run() {
  const value = this.get()  // 重新求值
  if (value !== this.value || isObject(value) || this.deep) {
    const oldValue = this.value
    this.value = value
    // 执行用户的回调函数
    if (this.user) {
      this.cb.call(this.vm, value, oldValue)
    }
  }
}
```

---

## 四、computed vs watch 完整对比

| 维度 | computed | watch |
|------|----------|-------|
| **本质** | 响应式属性（getter） | 观察者（回调函数） |
| **返回值** | 有返回值 | 无返回值（执行副作用） |
| **缓存** | 有缓存（dirty 标志） | 无缓存 |
| **异步** | 不支持异步 | 支持异步操作 |
| **执行时机** | 被读取时惰性计算 | 数据变化时立即执行 |
| **适用场景** | 派生状态、模板中的计算值 | 异步操作、开销较大的操作 |
| **Watcher 类型** | computed watcher（lazy） | user watcher（user） |

---

## 五、deep 选项的实现原理

### 5.1 作用

监听对象内部值的变化（深层嵌套属性）。

```js
// 不使用 deep
watch: {
  obj(newVal) {
    // 只监听 obj 引用变化，obj.a = 1 不会触发
  }
}

// 使用 deep
watch: {
  obj: {
    handler(newVal) { /* ... */ },
    deep: true  // obj.a = 1 也会触发
  }
}
```

### 5.2 实现原理

```js
// deep 的实现：递归访问所有属性，触发 getter 收集依赖
get() {
  pushTarget(this)  // 设置 Dep.target
  const value = this.getter.call(vm, vm)
  if (this.deep) {
    traverse(value)  // 递归访问所有属性
  }
  popTarget()
  return value
}

// traverse 的简化实现
function traverse(val) {
  const seen = new Set()
  _traverse(val, seen)
}

function _traverse(val, seen) {
  if (!isObject(val)) return
  // 避免循环引用
  if (seen.has(val)) return
  seen.add(val)
  // 遍历所有属性，触发 getter
  for (const key in val) {
    _traverse(val[key], seen)  // 递归
  }
}
```

**原理总结：**
- `traverse` 递归访问对象的每个属性
- 访问属性时触发 getter，将当前 watcher（Dep.target）收集到该属性的 Dep 中
- 这样任意深层属性变化都能通知到 watcher

---

## 六、immediate 选项

### 6.1 作用

创建 watcher 时立即执行一次回调，此时回调的参数为当前值（没有旧值）。

```js
watch: {
  name: {
    handler(newVal) {
      // immediate: true 时，创建时立即执行一次
      // 此时 newVal 是当前值，没有旧值
      console.log(newVal)
    },
    immediate: true
  }
}
```

### 6.2 实现原理

```js
// 在创建 watcher 后，立即调用回调
if (options.immediate) {
  // 此时 watcher.value 已经是当前值
  cb.call(vm, watcher.value)
}
```

---

## 七、回答问题

### Q1：computed 和 watch 在使用场景上有什么区别？

**computed：派生状态**
```js
computed: {
  // 适合：基于已有状态计算新值
  fullName() { return this.firstName + this.lastName },
  filteredList() { return this.list.filter(item => item.active) },
  totalPrice() { return this.items.reduce((sum, item) => sum + item.price, 0) }
}
```

**watch：副作用操作**
```js
watch: {
  // 适合：异步操作、开销大的操作、命令式操作
  searchText(val) {
    this.fetchSearchResults(val)  // API 请求
  },
 $route(to) {
    this.loadPageData(to.params.id)  // 路由变化时加载数据
  },
  selectedId(val) {
    localStorage.setItem('selected', val)  // 本地存储
  }
}
```

**选择原则：**
- 需要一个**值** → 用 `computed`
- 需要执行**动作** → 用 `watch`

---

### Q2：computed 的缓存机制是怎么实现的？为什么它"有缓存"而 methods 没有？

**核心：dirty 标志位**

```
computed watcher 初始化
  lazy: true, dirty: true（需要计算）

首次读取 computed 属性
  ↓
访问 getter → dirty === true
  ↓
执行计算函数，收集依赖
  ↓
dirty = false（标记为已缓存）
  ↓
再次读取 computed 属性
  ↓
dirty === false → 直接返回缓存值，跳过计算
  ↓
依赖数据变化 → 通知 watcher → dirty = true
  ↓
下次读取时 dirty === true → 重新计算
```

**为什么 methods 没有缓存？**
- `methods` 是函数调用语义，每次 `this.method()` 都会执行函数体
- `computed` 是属性访问语义，底层是 `Object.defineProperty` 的 getter
- getter 中有 `dirty` 标志控制是否需要重新执行
- 这个机制保证了：**依赖不变时，多次访问同一个 computed 只计算一次**

---

### Q3：watch 的 deep 和 immediate 选项分别做什么？deep 是怎么实现的？

**deep：深度监听**
```js
// deep: true 时，对象内部属性变化也能触发回调
watch: {
  user: {
    handler(newVal) { console.log('user changed') },
    deep: true
  }
}
this.user.name = 'new'  // 触发回调
```

**deep 实现原理：**
```
创建 watcher 时
  ↓
调用 getter（求值）→ 收集对 user 的依赖
  ↓
this.deep === true → 调用 traverse(value)
  ↓
递归访问 user 的每个属性（user.name、user.age、user.address.city...）
  ↓
每个属性的 getter 被触发 → 将当前 watcher 收集到每个属性的 Dep 中
  ↓
任意深层属性变化 → 通知 watcher → 执行回调
```

**immediate：立即执行**
```js
// immediate: true 时，创建 watcher 后立即执行一次回调
watch: {
  searchText: {
    handler(val) { this.fetchData(val) },
    immediate: true  // 创建时就执行一次 fetchData(this.searchText)
  }
}
```

**immediate 实现原理：**
```js
// 创建 watcher 后，立即用当前值调用回调
if (options.immediate) {
  cb.call(vm, watcher.value)  // watcher.value 是当前值
}
```
