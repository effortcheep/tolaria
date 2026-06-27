---
type: review
---
# Vue 2 的 $set 和 $delete

## 一、核心概念：Vue 2 的响应式局限

Vue 2 使用 `Object.defineProperty` 递归地把对象的每个属性转为 getter/setter，从而实现响应式。但这个方案有**根本性限制**——它只能拦截**已存在的属性的读写**，无法感知**属性的新增和删除**。

### `Object.defineProperty` 的工作方式

```js
// Vue 2 响应式的核心伪代码
function defineReactive(obj, key, val) {
  const dep = new Dep() // 每个属性一个依赖收集器
  Object.defineProperty(obj, key, {
    get() {
      dep.depend() // 收集依赖（谁在读这个属性）
      return val
    },
    set(newVal) {
      if (newVal === val) return
      val = newVal
      dep.notify() // 通知更新（这个属性变了）
    }
  })
}
```

关键点：**getter/setter 是定义在属性描述符上的**，没有属性 → 没有描述符 → 没有 getter/setter → 没有依赖收集和变更通知。

### `Observer` 类的初始化过程

```js
// src/core/observer/index.js 简化
class Observer {
  constructor(value) {
    this.value = value
    this.dep = new Dep() // 对象本身的 dep（用于 $set 等操作）
    def(value, '__ob__', this) // 标记已被观察

    if (Array.isArray(value)) {
      // 数组：重写 7 个变异方法
      protoAugment(value, arrayMethods)
      this.observeArray(value)
    } else {
      // 对象：遍历每个属性，转为 getter/setter
      this.walk(value)
    }
  }

  walk(obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }
}
```

注意 `walk` 遍历的是 `Object.keys()`——只遍历**已有属性**。之后新增的属性不在遍历范围内。

---

## 二、哪些情况数据变了但视图不更新？

### 情况 1：对象新增属性

```js
export default {
  data() {
    return {
      user: { name: 'Tom' }
    }
  },
  methods: {
    addAge() {
      this.user.age = 25 // ❌ 视图不更新
      // age 属性从未被 defineReactive 处理过
      // 没有 getter/setter → 没有 dep → 无法通知
    }
  }
}
```

### 情况 2：对象删除属性

```js
delete this.user.name // ❌ 视图不更新
// delete 操作符不会触发 setter
// 属性被移除了，但 Vue 不知道
```

### 情况 3：数组通过索引设置元素

```js
this.arr[0] = 'new value' // ❌ 视图不更新
// Vue 2 没有对数组索引设置 getter/setter
// 原因：性能代价太大（数组可能有几千个元素）
```

### 情况 4：修改数组的 length

```js
this.arr.length = 0 // ❌ 视图不更新
// length 不是通过 getter/setter 拦截的
```

### 为什么数组变异方法（push/pop/splice 等）可以触发更新？

Vue 2 重写了数组的 7 个变异方法：

```js
const methodsToPatch = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']

methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator(...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__

    // push/unshift/splice 可能插入新元素，需要观察
    let inserted
    if (method === 'push' || method === 'unshift') inserted = args
    else if (method === 'splice') inserted = args.slice(2)
    if (inserted) ob.observeArray(inserted)

    ob.dep.notify() // 手动触发更新
    return result
  })
})
```

变异方法内部调用了 `ob.dep.notify()`，直接通知了数组的 `__ob__` 上的依赖，所以能触发更新。

---

## 三、`$set` 的用法和解决方式

### 语法

```js
Vue.set(target, key, value)
// 或
this.$set(target, key, value)
```

### 解决对象新增属性的问题

```js
// ❌ 错误：视图不更新
this.user.age = 25

// ✅ 正确：使用 $set
this.$set(this.user, 'age', 25)
```

### 解决数组索引设置的问题

```js
// ❌ 错误
this.arr[0] = 'new'

// ✅ 正确
this.$set(this.arr, 0, 'new')
// 等价于
this.arr.splice(0, 1, 'new')
```

### `$delete` 的用法

```js
// ❌ 错误：视图不更新
delete this.user.name

// ✅ 正确
this.$delete(this.user, 'name')
// 或
Vue.delete(this.user, 'name')
```

---

## 四、`$set` 的底层实现原理

源码位于 `src/core/observer/index.js`：

```js
export function set(target, key, val) {
  // 1. target 必须是对象或数组，不能是原始值
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${target}`)
  }

  // 2. 如果 target 是数组，且 key 是合法索引
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    // 用 splice 替代直接赋值（splice 已被重写，能触发更新）
    target.splice(key, 1, val)
    return val
  }

  // 3. 如果 key 已经存在于 target 上，直接赋值（触发已有的 setter）
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }

  // 4. 以下是"新增属性"的核心逻辑
  const ob = target.__ob__ // 获取 Observer 实例

  // 5. target 不是响应式对象，直接赋值即可
  if (!ob) {
    target[key] = val
    return val
  }

  // 6. ★ 核心：对新属性定义响应式（调用 defineReactive）
  defineReactive(ob.value, key, val)

  // 7. ★ 核心：手动触发依赖通知
  ob.dep.notify()

  return val
}
```

### 执行流程图解

```
$set(user, 'age', 25)
│
├─ target 是数组？
│  └─ 是 → splice(key, 1, val) → 触发数组变异方法 → 更新
│
├─ key 已存在？
│  └─ 是 → target[key] = val → 触发已有 setter → 更新
│
└─ key 不存在（新增属性）
   ├─ target.__ob__ 存在？
   │  ├─ 否 → 直接赋值，不做响应式处理
   │  └─ 是 → 继续 ↓
   │
   ├─ defineReactive(ob.value, key, val)
   │  └─ 为新属性创建 getter/setter + 新的 dep
   │
   └─ ob.dep.notify()
      └─ 通知对象级别的 watcher 更新
```

### 为什么需要 `ob.dep.notify()`？

`defineReactive` 只是给新属性建立了响应式，但此时还没有任何 watcher 订阅这个新属性的 dep。需要通过**对象本身的 `__ob__.dep`** 来通知所有依赖这个对象的 watcher：

```js
// defineReactive 内部
function defineReactive(obj, key, val) {
  const dep = new Dep()  // 新属性的 dep（此时是空的，没人订阅）
  Object.defineProperty(obj, key, {
    get() {
      dep.depend()
      // 同时也让 __ob__.dep 收集了当前 watcher
      // 因为在 observe() 时访问了 __ob__，触发了 __ob__.dep.depend()
      return val
    },
    set(newVal) {
      val = newVal
      dep.notify() // 后续修改新属性，走这条路径
    }
  })
}
```

`ob.dep.notify()` 保证了：即使新属性的 dep 还没有 watcher 订阅，依赖这个**对象**的 watcher（比如模板中遍历了这个对象的所有 key）也能收到通知。

---

## 五、`$delete` 的实现原理

```js
export function del(target, key) {
  // 1. 同样不能操作原始值
  if (isUndef(target) || isPrimitive(target)) {
    warn(...)
  }

  // 2. 数组：用 splice 删除（触发变异方法）
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }

  // 3. 获取 Observer 实例
  const ob = target.__ob__

  // 4. target 上没有这个属性，直接返回
  if (!target.hasOwnProperty(key)) return

  // 5. 删除属性
  delete target[key]

  // 6. 如果不是响应式对象，到此结束
  if (!ob) return

  // 7. 手动通知依赖更新
  ob.dep.notify()
}
```

`$delete` 的逻辑与 `$set` 对称：删除属性后，通过 `__ob__.dep.notify()` 通知更新。

---

## 六、Vue 3 如何解决这个问题？

Vue 3 使用 `Proxy` 代替 `Object.defineProperty`：

```js
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    track(target, key) // 收集依赖
    return Reflect.get(target, key, receiver)
  },
  set(target, key, value, receiver) {
    const result = Reflect.set(target, key, value, receiver)
    trigger(target, key) // 触发更新
    return result
  },
  deleteProperty(target, key) {
    const result = Reflect.deleteProperty(target, key)
    trigger(target, key) // 删除也能触发更新
    return result
  }
})
```

`Proxy` 拦截的是**对象层面的操作**，而不是属性层面的描述符，所以新增属性、删除属性、数组索引修改都能自然拦截，不需要 `$set` / `$delete`。

---

## 七、回答问题

### ① 什么情况下数据变了但视图不更新？`$set` 怎么解决？

**4 种情况：**

| 场景 | 原因 | 解决方式 |
| --- | --- | --- |
| 对象新增属性 | `Object.defineProperty` 只能拦截已有属性，新属性没有 getter/setter | `this.$set(obj, key, val)` |
| 对象删除属性 | `delete` 不触发 setter | `this.$delete(obj, key)` |
| 数组索引赋值 | 未对数组索引做 getter/setter（性能考虑） | `this.$set(arr, index, val)` 或 `arr.splice(index, 1, val)` |
| 数组修改 length | length 未被拦截 | `arr.splice(newLength)` |

**`$set` 的解决方式：**
1. 数组场景 → 内部转为 `splice` 操作（已被重写，能触发更新）
2. 属性已存在 → 直接赋值，走已有的 setter
3. 属性不存在 → 调用 `defineReactive` 为新属性建立 getter/setter，再通过 `__ob__.dep.notify()` 通知对象级别的 watcher 更新

### ② `$set` 的底层实现原理是什么？

核心步骤：

1. **数组**：转为 `splice` 调用，利用重写的变异方法触发更新
2. **属性已存在**：直接赋值，触发已有 setter
3. **属性不存在（新增）**：
   - 通过 `target.__ob__` 获取 Observer 实例
   - 调用 `defineReactive(ob.value, key, val)` 为新属性创建响应式（getter/setter + dep）
   - 调用 `ob.dep.notify()` 通过对象级别的 dep 通知所有依赖该对象的 watcher

关键理解：`$set` 本质上是**手动补上了 Vue 初始化时遗漏的响应式定义**，并通过对象自身的 dep（而非新属性的 dep）来触发更新，因为新属性的 dep 此时还没有 watcher 订阅。
