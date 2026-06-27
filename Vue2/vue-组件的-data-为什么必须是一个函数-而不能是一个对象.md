---
type: review
---

# Vue 组件的 data 为什么必须是一个函数，而不能是一个对象？

## 一、核心结论

| 场景 | data 的形式 | 原因 |
|------|-------------|------|
| **根实例**（`new Vue()`） | 可以是对象 | 根实例只有一个，不存在共享问题 |
| **组件** | **必须是函数** | 组件可能被多次复用，需要返回独立的数据副本 |

---

## 二、根本原因：引用类型共享问题

### 1. 对象是引用类型

JavaScript 中对象是引用类型，多个变量指向同一个对象时，修改其中一个会影响所有引用：

```js
const obj = { count: 0 }
const a = obj
const b = obj

a.count = 1
console.log(b.count) // 1 —— b 也受影响
```

### 2. 如果组件 data 是对象

```js
// 假设 data 是对象
const MyComponent = {
  data: {
    count: 0
  }
}
```

当这个组件被**多次使用**时，所有实例的 `this.$data` 都指向**同一个对象**：

```html
<MyComponent />  <!-- 实例1: this.$data → { count: 0 } -->
<MyComponent />  <!-- 实例2: this.$data → 同一个对象 -->
<MyComponent />  <!-- 实例3: this.$data → 同一个对象 -->
```

一个实例修改 `count`，所有实例的 `count` 都会变——**状态污染**。

### 3. 如果组件 data 是函数

```js
const MyComponent = {
  data() {
    return {
      count: 0
    }
  }
}
```

每次创建组件实例时，都会调用 `data()` 返回一个**全新的对象**：

```html
<MyComponent />  <!-- 实例1: this.$data → { count: 0 } (新对象) -->
<MyComponent />  <!-- 实例2: this.$data → { count: 0 } (新对象) -->
<MyComponent />  <!-- 实例3: this.$data → { count: 0 } (新对象) -->
```

每个实例拥有**独立的数据副本**，互不影响。

---

## 三、源码层面的验证

### Vue 2 源码中的 mergeDataStrategies

```js
// src/core/util/options.js
strategies.data = function (parentVal, childVal, vm) {
  if (!vm) {
    // 组件：data 必须是函数
    if (childVal && typeof childVal !== 'function') {
      warn(
        'The "data" option should be a function ' +
        'that returns a per-component value in component definitions.',
        vm
      )
      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }
  // 根实例：可以是对象或函数
  return mergeDataOrFn(parentVal, childVal, vm)
}
```

### initData 中的调用

```js
// src/core/instance/state.js
function initData(vm) {
  let data = vm.$options.data
  // 如果 data 是函数，调用它获取返回值
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
}

function getData(data, vm) {
  // 调用 data 函数，绑定 this 为组件实例
  return data.call(vm, vm)
}
```

**关键点**：Vue 内部会判断 `data` 是否为函数，如果是函数就调用它获取返回值。

---

## 四、根实例为什么可以用对象？

```js
// 根实例只创建一次，不存在复用问题
new Vue({
  data: { count: 0 }  // ✓ 可以是对象
})
```

但 Vue 仍然**推荐根实例也使用函数形式**，保持一致性：

```js
new Vue({
  data() {
    return { count: 0 }  // ✓ 推荐
  }
})
```

---

## 五、Vue 3 的变化

### Vue 3 中组件 data 仍然是函数

```js
// Vue 3 Options API —— data 必须是函数
export default {
  data() {
    return { count: 0 }
  }
}
```

规则没有变化，原因也一样：避免多个组件实例共享同一份数据。

### Vue 3 Composition API 中无此限制

```js
// Composition API 使用 ref/reactive，天然独立
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)  // 每次 setup() 调用都创建新的 ref
    return { count }
  }
}
```

`setup()` 函数本身就是每次实例化时调用的，所以不存在共享问题。

---

## 六、常见追问

### Q1：函数返回的对象还是引用类型，为什么不会共享？

**关键在于"何时调用"**：

```js
data() {
  return { count: 0 }  // 每次调用都 return 新对象
}
```

- `data()` 函数**每次被调用**都会执行 `return { count: 0 }`
- `return { count: 0 }` 每次执行都会创建一个**全新的对象**
- 所以每个实例拿到的是不同的对象

### Q2：如果 data 函数返回的对象内部嵌套了对象，嵌套对象会共享吗？

```js
data() {
  return {
    user: { name: 'Alice' }  // user 对象只创建一次
  }
}
```

**不会共享**，因为每次调用 `data()` 都会创建新的 `user` 对象。

但如果是**同一个组件的多个实例**，每个实例的 `data()` 调用是独立的，所以嵌套对象也是独立的。

### Q3：为什么 Vuex 的 state 可以是对象？

Vuex 的 Store 是**单例模式**，整个应用只有一个 Store 实例，不存在复用问题：

```js
// Vuex —— 单例
const store = new Vuex.Store({
  state: { count: 0 }  // 只有一个实例，可以是对象
})
```

### Q4：为什么 Vue 不自动把对象包装成函数？

**设计哲学**：显式优于隐式

- 让开发者明确知道 `data` 是函数，每次返回新对象
- 避免隐式行为带来的困惑
- 与 JavaScript 的函数作用域语义一致

---

## 七、总结

| 问题 | 答案 |
|------|------|
| 为什么组件 data 必须是函数？ | 避免多个组件实例共享同一份数据，导致状态污染 |
| 核心原理 | 函数每次调用返回新对象，每个实例拥有独立数据副本 |
| 根实例为什么可以用对象？ | 根实例只有一个，不存在复用问题 |
| Vue 3 有变化吗？ | Options API 规则不变；Composition API 无此限制 |
