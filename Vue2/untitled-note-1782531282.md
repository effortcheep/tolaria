---
type: review
---

# Vue 2 的数据双向绑定原理

两个小问：\
**① Vue 2 的响应式系统是怎么实现的？从 data 中的数据变化到视图更新，整个流程是怎样的？**\
**② 所谓的"双向绑定"，v-model 是怎么实现双向的？它和 v-bind + v-on 有什么关系？**

---

## 一、响应式系统的核心：Object.defineProperty

Vue 2 的响应式基于 `Object.defineProperty()` 为 data 中**每个属性**定义 getter/setter。

```js
// Vue 2 源码简化版
function defineReactive(obj, key, val) {
  const dep = new Dep() // 每个属性一个依赖收集器

  Object.defineProperty(obj, key, {
    get() {
      if (Dep.target) {
        dep.depend() // 收集依赖
      }
      return val
    },
    set(newVal) {
      if (newVal === val) return
      val = newVal
      dep.notify() // 通知更新
    }
  })
}
```

**核心机制：**
- **getter 中收集依赖** — 谁读取了这个数据，就把谁记录下来
- **setter 中通知更新** — 数据变化时，通知所有依赖者重新计算/渲染

---

## 二、三大核心概念

### 1. Observer（响应式转换器）

递归遍历 data 对象的所有属性，对每个属性调用 `defineReactive`。

```js
class Observer {
  constructor(value) {
    this.walk(value)
  }
  walk(obj) {
    for (const key in obj) {
      defineReactive(obj, key, obj[key])
    }
  }
}
```

- 对象类型 → 递归遍历每个属性
- 数组类型 → 重写 7 个变异方法（push/pop/shift/unshift/splice/sort/reverse）

### 2. Dep（依赖收集器）

每个响应式属性都有一个 Dep 实例，负责管理所有依赖该属性的 Watcher。

```js
class Dep {
  constructor() {
    this.subs = [] // 订阅者列表
  }
  addSub(watcher) {
    this.subs.push(watcher)
  }
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this) // 让 Watcher 记录这个 Dep
    }
  }
  notify() {
    this.subs.forEach(watcher => watcher.update())
  }
}

Dep.target = null // 全局唯一，当前正在求值的 Watcher
```

### 3. Watcher（观察者/订阅者）

连接数据和视图的桥梁。每个组件渲染、computed、watch 都对应一个 Watcher。

```js
class Watcher {
  constructor(vm, getter) {
    this.vm = vm
    this.getter = getter
    this.deps = []
    this.get() // 立即求值，触发依赖收集
  }
  get() {
    Dep.target = this       // 1. 将自己设为全局 target
    this.getter.call(this.vm) // 2. 执行求值，触发 getter → 收集依赖
    Dep.target = null       // 3. 清空 target
  }
  addDep(dep) {
    this.deps.push(dep)
    dep.addSub(this) // 双向记录
  }
  update() {
    queueWatcher(this) // 加入异步更新队列
  }
}
```

---

## 三、完整响应式流程

### 阶段一：初始化（数据劫持）

```
new Vue()
  → initState()
    → initData()
      → observe(data)
        → new Observer(data)
          → walk(data)
            → defineReactive(obj, key, val)  // 为每个属性设置 getter/setter
              → new Dep()                     // 每个属性创建一个 Dep
```

**结果：** data 中每个属性都有了 getter/setter，每个属性都有自己的 Dep。

### 阶段二：模板编译（依赖收集）

```
$mount()
  → 编译模板，生成渲染函数
    → 创建渲染 Watcher
      → Watcher.get()
        → Dep.target = 渲染 Watcher
        → 执行渲染函数
          → 访问 this.message  → 触发 getter
            → dep.depend()
              → 渲染 Watcher 记录 dep
              → dep.subs 中加入渲染 Watcher
        → Dep.target = null
```

**结果：** 渲染过程中访问了哪些数据，就和哪些 Dep 建立了双向关联。

### 阶段三：数据更新（派发更新）

```
this.message = '新值'
  → 触发 setter
    → dep.notify()
      → 遍历 dep.subs（所有 Watcher）
        → watcher.update()
          → queueWatcher(watcher)  // 加入异步队列
            → nextTick()
              → flushQueue()
                → watcher.run()
                  → 重新执行渲染函数
                    → 生成新的 VNode
                      → patch(oldVNode, newVNode)
                        → 更新真实 DOM
```

**结果：** 数据变化 → setter 触发 → 通知 Watcher → 异步更新 DOM。

---

## 四、依赖收集的精妙设计

### 为什么需要 Dep.target 全局变量？

同一时刻**只有一个 Watcher 在求值**。通过全局变量 `Dep.target` 确定"当前是谁在读取数据"。

```
Watcher 求值开始
  → Dep.target = 自己
  → 执行求值（触发 getter）
  → getter 中 dep.depend() 检查 Dep.target
  → 有值 → 建立关联
  → 求值结束
  → Dep.target = null
```

### 一个数据多个 Watcher

一个响应式属性可以被多个 Watcher 依赖（多个组件使用同一个数据）：

```
this.message
  → 被组件 A 的渲染 Watcher 依赖
  → 被组件 B 的渲染 Watcher 依赖
  → 被 computed 属性 C 的 Watcher 依赖

message 变化时 → dep.notify() → 三个 Watcher 都收到通知
```

### 一个 Watcher 多个 Dep

一个 Watcher（如渲染 Watcher）可以依赖多个响应式属性：

```
渲染函数访问了 this.name 和 this.age
  → name 的 dep 记录了渲染 Watcher
  → age 的 dep 记录了渲染 Watcher
  → 渲染 Watcher 的 deps 中有 [name的dep, age的dep]
```

---

## 五、数组的特殊处理

`Object.defineProperty` **无法检测数组索引变化和长度变化**。Vue 2 采用**重写原型方法**的方案：

```js
// 获取数组原始原型
const arrayProto = Array.prototype
const arrayMethods = Object.create(arrayProto)

// 重写 7 个变异方法
const methodsToPatch = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']

methodsToPatch.forEach(method => {
  arrayMethods[method] = function(...args) {
    const result = arrayProto[method].apply(this, args) // 执行原方法
    const ob = this.__ob__  // 获取 Observer 实例
    ob.dep.notify()         // 手动通知更新
    return result
  }
})
```

**为什么能触发更新？**
- 重写的方法在执行原方法后，手动调用 `dep.notify()`
- 新插入的元素（如 push 的参数、splice 插入的元素）会被 `observeArray` 再次观测

**限制：**
- `this.arr[0] = '新值'` → **不能检测**
- `this.arr.length = 0` → **不能检测**
- 必须用 `this.$set(this.arr, 0, '新值')` 或 `this.arr.splice(0, 1, '新值')`

---

## 六、Vue.set / this.$set 的必要性

### 为什么需要？

`Object.defineProperty` **只能劫持已存在的属性**。新增属性时，新属性没有经过 `defineReactive`，因此不是响应式的。

```js
// 问题场景
data() {
  return { user: { name: '张三' } }
}
// 某处新增属性
this.user.age = 25  // ❌ 不是响应式的，视图不会更新

// 正确做法
this.$set(this.user, 'age', 25)  // ✅ 响应式，触发视图更新
```

### $set 做了什么？

```js
Vue.set = function(target, key, val) {
  // 1. 如果 target 是响应式的，定义响应式属性
  defineReactive(target, key, val)
  // 2. 手动触发通知
  target.__ob__.dep.notify()
}
```

---

## 七、v-model 的本质

### 1. v-model 在原生 HTML 元素上

`v-model` 是**语法糖**，等价于 `v-bind` + `v-on`：

```html
<!-- 这两种写法完全等价 -->
<input v-model="message">

<input :value="message" @input="message = $event.target.value">
```

拆解：
- `:value="message"` — **数据 → 视图**（单向绑定，数据驱动视图）
- `@input="message = $event.target.value"` — **视图 → 数据**（事件监听，回写数据）

不同表单元素的对应关系：

| 元素 | v-bind | v-on |
|------|--------|------|
| `<input type="text">` | `:value` | `@input` |
| `<input type="checkbox">` | `:checked` | `@change` |
| `<input type="radio">` | `:checked` | `@change` |
| `<select>` | `:value` | `@change` |
| `<textarea>` | `:value` | `@input` |

### 2. v-model 在组件上

```html
<!-- 父组件 -->
<MyInput v-model="message">

<!-- 等价于 -->
<MyInput :value="message" @input="message = $event">
```

子组件内部：
```js
// MyInput.vue
export default {
  props: ['value'],
  methods: {
    handleInput(e) {
      this.$emit('input', e.target.value) // 触发父组件的 input 事件
    }
  }
}
```

### 3. "双向绑定"的真相

**数据是单向流动的**，v-model 只是让单向数据流的书写更简洁：

```
单向数据流（本质）：
  data → props → 子组件视图
  子组件事件 → $emit → 父组件修改 data

v-model（语法糖）：
  把上面两步合并成一个 v-model="xxx"
```

---

## 八、v-model vs v-bind + v-on

| 维度 | v-model | v-bind + v-on |
|------|---------|---------------|
| 本质 | 语法糖 | 底层写法 |
| 代码量 | 一行 | 两行 |
| 可读性 | 更直观 | 更明确 |
| 灵活性 | 只能绑定一个值 | 可自定义事件名、处理逻辑 |
| 多个 v-model | Vue 2 不支持 | 可以用多个 prop + event |
| 适用场景 | 表单双向绑定 | 自定义组件通信 |

**总结：v-model 不是真正的"双向绑定"，而是"单向数据流 + 事件回写"的语法糖。**

---

## 九、Vue 2 响应式的局限性总结

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 新增属性不响应 | `defineProperty` 只劫持已有属性 | `$set` |
| 删除属性不响应 | 同上 | `$delete` |
| 数组索引赋值不响应 | `defineProperty` 不检测索引 | `$set` 或 `splice` |
| 数组长度变化不响应 | `defineProperty` 不检测 length | `splice` |
| 需要递归遍历 | 初始化时深度遍历所有属性 | 惰性代理（Vue 3） |
| 性能开销 | 每个属性都创建 Dep/defineProperty | Proxy 整体代理（Vue 3） |

---

## 回答部分

### ① Vue 2 的响应式系统是怎么实现的？从 data 中的数据变化到视图更新，整个流程是怎样的？

**核心原理：** `Object.defineProperty` 为 data 每个属性定义 getter/setter，在 getter 中收集依赖，在 setter 中通知更新。

**三大核心：**
- **Observer** — 递归遍历 data，为每个属性设置 getter/setter
- **Dep** — 每个属性一个依赖收集器，管理所有 Watcher
- **Watcher** — 连接数据和视图的桥梁（渲染 Watcher、computed Watcher、watch Watcher）

**完整流程：**

```
【初始化】new Vue → observe(data) → Observer → defineReactive(每个属性) → 创建 Dep

【依赖收集】$mount → 编译模板 → 创建渲染 Watcher
  → Dep.target = Watcher → 执行渲染函数
  → 访问数据 → 触发 getter → dep.depend() → Watcher 与 Dep 双向关联

【派发更新】修改数据 → 触发 setter → dep.notify()
  → 所有 Watcher.update() → queueWatcher → nextTick
  → flushQueue → watcher.run() → 重新渲染 → patch → 更新 DOM
```

**一句话：** 数据被读取时收集依赖（谁用了我），数据变化时通知依赖者（我变了，你更新）。

---

### ② 所谓的"双向绑定"，v-model 是怎么实现双向的？它和 v-bind + v-on 有什么关系？

**v-model 是语法糖**，等价于 `v-bind` + `v-on` 的组合：

```html
<!-- 完全等价 -->
<input v-model="message">
<input :value="message" @input="message = $event.target.value">
```

**"双向"拆解为两个单向：**
- **数据 → 视图：** `:value="message"` — 数据变化 → 触发 setter → 依赖更新 → 重新渲染 → DOM 的 value 更新
- **视图 → 数据：** `@input` — 用户输入触发事件 → 事件处理函数修改 `message` → 触发 setter → 视图更新

**本质：数据永远是单向流动的。** v-model 只是把"数据绑定 + 事件监听"合并成一个指令，让表单处理代码更简洁。不是真正的双向数据流，而是单向数据流 + 事件回写的语法糖。

**在组件上：** `v-model` 等价于传递 `value` prop + 监听 `input` 事件，子组件通过 `$emit('input', newVal)` 回写数据。
