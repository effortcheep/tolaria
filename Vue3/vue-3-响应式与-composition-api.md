---
type: review
---
# Vue 3 响应式与 Composition API

## 一、Vue 3 响应式基础

### 1. Proxy 响应式原理

Vue 3 使用 `Proxy` 替代 Vue 2 的 `Object.defineProperty`：

```js
// Vue 3 核心实现（简化）
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)  // 依赖收集
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver)
      trigger(target, key)  // 触发更新
      return result
    }
  })
}
```

**Proxy 优势：**
- 能监听属性的**新增/删除**（Vue 2 需要 `Vue.set`）
- 能监听**数组索引**和 `length` 变化
- 支持 `Map`、`Set`、`WeakMap`、`WeakSet`
- 不需要递归初始化，性能更好

### 2. ref 与 reactive 的实现

```
ref(value)
  ├── 基本类型 → 包装为 { value: ... } 对象，通过 .value 访问
  └── 对象类型 → 内部调用 reactive 进行深度响应式转换

reactive(obj)
  └── 直接返回 Proxy 代理对象，深度响应式
```

---

## 二、ref 与 reactive 深入对比

| 对比维度 | `ref` | `reactive` |
|---------|-------|-----------|
| **接受参数** | 任意值（基本类型 + 对象） | 只能是对象/数组 |
| **返回值** | `{ value: T }` 的包装对象 | Proxy 代理对象 |
| **访问方式** | 需要 `.value` | 直接访问属性 |
| **模板使用** | 自动解包，无需 `.value` | 直接使用 |
| **解构** | 不影响响应性 | 解构后丢失响应性 |
| **重新赋值** | 可以整体替换 `.value` | 不能整体替换 |
| **底层实现** | `RefImpl` 类 | `Proxy` 代理 |

### ref 的自动解包

```js
const count = ref(0)

// 在模板中自动解包
// <template>{{ count }}</template>  ✅ 不需要 .value

// 在 reactive 对象中自动解包
const state = reactive({ count })
console.log(state.count)  // 0，自动解包，不需要 .value

// 在数组中不会自动解包
const arr = ref([ref(1)])
console.log(arr.value[0].value)  // 需要 .value
```

### reactive 的局限性

```js
let state = reactive({ count: 0 })

// ❌ 问题1：不能整体替换
state = { count: 1 }  // 会丢失响应性

// ✅ 解决：使用 Object.assign
Object.assign(state, { count: 1 })

// ❌ 问题2：解构丢失响应性
const { count } = state  // count 变成普通数字

// ✅ 解决：使用 toRefs
const { count } = toRefs(state)  // count 变成 ref
```

---

## 三、选择 ref 还是 reactive 的原则

### 官方推荐

> **推荐使用 `ref`** 作为声明响应式状态的主要 API。— Vue 3 官方文档

### 使用场景决策

```
声明响应式状态
│
├── 基本类型（string/number/boolean）→ 必须用 ref
│
└── 对象/数组
    ├── 需要整体替换 → ref
    ├── 不需要整体替换 → reactive 或 ref 都可以
    ├── 需要解构 → ref 或 reactive + toRefs
    └── 组合函数返回值 → 推荐 ref（统一 API）
```

### 实际开发建议

```js
// ✅ 推荐：统一使用 ref
const count = ref(0)
const user = ref({ name: 'Alice' })
const list = ref([1, 2, 3])

// ✅ 也可以：简单组件内使用 reactive
const form = reactive({
  username: '',
  password: ''
})

// ✅ 组合函数返回 ref
function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }  // 返回 ref，调用方统一用 .value
}
```

---

## 四、watch 与 watchEffect 深入对比

### watch

```js
// 基本用法
const count = ref(0)

watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`)
})

// 监听 reactive 对象的某个属性
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (newVal, oldVal) => { /* ... */ }
)

// 监听多个源
watch(
  [count, () => state.name],
  ([newCount, newName], [oldCount, oldName]) => { /* ... */ }
)

// 深度监听
const user = ref({ name: 'Alice', age: 25 })
watch(user, (newVal) => { /* ... */ }, { deep: true })

// 立即执行
watch(count, (newVal) => { /* ... */ }, { immediate: true })
```

### watchEffect

```js
const count = ref(0)
const name = ref('Alice')

// 自动追踪依赖
watchEffect(() => {
  console.log(`count: ${count.value}, name: ${name.value}`)
  // 首次立即执行
  // count 或 name 变化时重新执行
})

// 清除副作用
watchEffect((onCleanup) => {
  const timer = setTimeout(() => { /* ... */ }, 1000)
  onCleanup(() => clearTimeout(timer))  // 下次执行前或卸载时调用
})

// 停止监听
const stop = watchEffect(() => { /* ... */ })
stop()  // 手动停止
```

### 核心区别对比表

| 对比维度 | `watch` | `watchEffect` |
|---------|---------|---------------|
| **依赖声明** | 显式指定监听源 | 自动追踪回调中用到的响应式依赖 |
| **惰性** | 默认惰性，首次不执行 | 立即执行，首次就运行 |
| **回调参数** | 有 `(newVal, oldVal)` | 无参数 |
| **访问旧值** | ✅ 可以 | ❌ 不行 |
| **适用场景** | 需要对比新旧值 | 只需要副作用，不需要旧值 |
| **清理副作用** | 支持 `onCleanup` | 支持 `onCleanup` |

---

## 五、computed 与 watch 深入对比

### computed

```js
const count = ref(0)

// 只读 computed
const double = computed(() => count.value * 2)
console.log(double.value)  // 0

// 可写 computed
const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(newFullName) {
    const [first, last] = newFullName.split(' ')
    firstName.value = first
    lastName.value = last
  }
})

// computed 是惰性求值的
// 只有依赖变化时才重新计算，有缓存
const heavy = computed(() => {
  console.log('计算中...')  // 依赖不变时不会执行
  return expensiveOperation(count.value)
})
```

### watch

```js
const count = ref(0)

// watch 执行副作用
watch(count, (newVal, oldVal) => {
  console.log(`变化: ${oldVal} → ${newVal}`)
  localStorage.setItem('count', newVal)  // 副作用
  fetch(`/api/data?id=${newVal}`)         // 异步操作
})

// watchEffect 执行副作用
watchEffect(() => {
  document.title = `Count: ${count.value}`  // 副作用
})
```

### 核心区别对比表

| 对比维度 | `computed` | `watch`/`watchEffect` |
|---------|------------|----------------------|
| **返回值** | ✅ 返回计算结果 | ❌ 无返回值（void） |
| **缓存** | ✅ 有缓存，依赖不变不重算 | ❌ 每次变化都执行 |
| **惰性** | ✅ 惰性，访问时才计算 | `watch` 惰性，`watchEffect` 立即 |
| **副作用** | ❌ 不应有副作用 | ✅ 专门用于副作用 |
| **同步/异步** | 必须同步 | 可以异步 |
| **用途** | 派生状态 | 执行操作/副作用 |

### 选择决策

```
需要基于响应式状态创建新值？
│
├── 是，需要返回值 → computed
│   ├── double = computed(() => count * 2)
│   └── filtered = computed(() => list.filter(...))
│
└── 否，需要执行副作用 → watch / watchEffect
    ├── 需要旧值对比 → watch
    ├── 需要精确控制触发 → watch
    └── 只需要自动追踪 → watchEffect
```

---

## 六、Composition API 其他重要概念

### 生命周期钩子

```js
import {
  onBeforeMount, onMounted,
  onBeforeUpdate, onUpdated,
  onBeforeUnmount, onUnmounted,
  onActivated, onDeactivated,
  onErrorCaptured
} from 'vue'

setup() {
  onMounted(() => { /* DOM 已挂载 */ })
  onUnmounted(() => { /* 清理副作用 */ })
}
```

### provide / inject（依赖注入）

```js
// 祖先组件
import { provide, ref } from 'vue'

setup() {
  const theme = ref('dark')
  provide('theme', theme)  // 提供响应式数据
}

// 后代组件
import { inject } from 'vue'

setup() {
  const theme = inject('theme', 'light')  // 注入，有默认值
}
```

### toRef 与 toRefs

```js
const state = reactive({ count: 0, name: 'Alice' })

// toRef：为某个属性创建 ref
const countRef = toRef(state, 'count')
countRef.value++  // 会同步修改 state.count

// toRefs：将所有属性转为 ref
const { count, name } = toRefs(state)
count.value++  // 会同步修改 state.count
```

---

## 七、问题回答

### Q1：ref 和 reactive 有什么区别？分别什么时候用？

**核心区别：**

| | `ref` | `reactive` |
|---|-------|-----------|
| 参数 | 任意值 | 仅对象/数组 |
| 访问 | `.value` | 直接访问 |
| 替换 | 可整体替换 `.value` | 不能整体替换 |
| 解构 | 不影响响应性 | 丢失响应性 |

**使用时机：**
- **基本类型** → 必须用 `ref`
- **需要整体替换的对象** → 用 `ref`
- **组合函数返回值** → 推荐 `ref`（统一 API）
- **简单组件内的表单对象** → 可用 `reactive`

**官方推荐：统一使用 `ref`。**

---

### Q2：watch 和 watchEffect 有什么区别？

| | `watch` | `watchEffect` |
|---|---------|---------------|
| 依赖 | 显式指定 | 自动追踪 |
| 首次执行 | 默认不执行 | 立即执行 |
| 旧值 | ✅ 能访问 | ❌ 不能 |
| 返回值 | 返回 stop 函数 | 返回 stop 函数 |

**选择：**
- 需要对比新旧值 → `watch`
- 只需要副作用，不想手动指定依赖 → `watchEffect`

---

### Q3：computed 和 watch 的区别是什么？什么场景用哪个？

| | `computed` | `watch` |
|---|------------|---------|
| 返回值 | ✅ 有 | ❌ 无 |
| 缓存 | ✅ 有 | ❌ 无 |
| 副作用 | ❌ 不应有 | ✅ 专门用于 |
| 异步 | ❌ 不支持 | ✅ 支持 |

**选择：**
- **派生新值**（如过滤列表、格式化数据） → `computed`
- **执行操作**（如发请求、写 localStorage、操作 DOM） → `watch` / `watchEffect`

```js
// ✅ computed：派生值
const filteredList = computed(() => list.value.filter(item => item.active))

// ✅ watch：副作用
watch(count, (val) => {
  localStorage.setItem('count', val)
  fetch(`/api/data/${val}`)
})
```
