---
type: review
---
# Pinia 的核心概念和使用方式

## 一、Vuex 的回顾与痛点

### Vuex 的核心概念

- **State**：单一状态树，存储全局状态
- **Getters**：派生状态，类似 computed
- **Mutations**：唯一修改 State 的方式，必须是同步函数
- **Actions**：处理异步操作，通过 commit 触发 Mutation
- **Modules**：将 Store 拆分为模块，支持命名空间

### Vuex 的痛点

1. **Mutation 多余**：必须通过 Mutation 修改 State，增加了样板代码，开发工具可以直接追踪 Action
2. **TypeScript 支持差**：字符串类型的 Mutation/Action 名称无法被 TS 推导，类型提示困难
3. **Modules 嵌套复杂**：命名空间嵌套后路径冗长（`user/profile/updateName`），dispatch 写法繁琐
4. **API 设计分裂**：`mapState`、`mapGetters`、`mapActions`、`mapMutations` 四个辅助函数，概念过多
5. **组合式 API 支持弱**：Vuex 4 对 Composition API 的支持不够原生

---

## 二、Pinia 的设计理念

### 核心思想

- **去中心化**：没有单一根 Store，每个 Store 独立定义、独立注册
- **扁平化**：取消 Modules 概念，用多个独立 Store 代替嵌套模块
- **简化 API**：去掉 Mutations，只剩 State / Getters / Actions
- **Composition API 友好**：Store 本质就是一个组合式函数

### 与 Vuex 的关系

- Pinia 是 Vuex 5 的官方实现（Vue 核心团队开发）
- 完全替代 Vuex，不再需要 Vuex
- Vue 官方文档已将 Pinia 推荐为唯一状态管理方案

---

## 三、Pinia 核心 API

### 1. `defineStore` — 定义 Store

两种写法：

**Option Store 写法**（类似 Options API）：
```js
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({
    count: 0,
    name: 'Counter'
  }),
  getters: {
    doubleCount: (state) => state.count * 2
  },
  actions: {
    increment() {
      this.count++
    }
  }
})
```

**Setup Store 写法**（类似 Composition API）：
```js
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const name = ref('Counter')
  const doubleCount = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  return { count, name, doubleCount, increment }
})
```

### 2. Store 实例 API

| API | 作用 |
|-----|------|
| `store.$reset()` | 将 State 重置为初始值（仅 Option Store） |
| `store.$patch({})` | 批量修改多个 State 属性 |
| `store.$patch((state) => {})` | 函数式批量修改（可处理数组等复杂操作） |
| `store.$subscribe((mutation, state) => {})` | 监听 State 变化 |
| `store.$onAction(({ name, args, after }) => {})` | 监听 Action 调用 |
| `store.$dispose()` | 销毁 Store，停止所有订阅 |

### 3. `storeToRefs` — 解构保持响应式

```js
import { storeToRefs } from 'pinia'

// ❌ 直接解构会丢失响应式
const { count, name } = useCounterStore()

// ✅ 使用 storeToRefs 保持响应式
const { count, name } = storeToRefs(useCounterStore())
// actions 直接解构即可
const { increment } = useCounterStore()
```

### 4. `mapState` / `mapStores` — 选项式 API 辅助函数

```js
import { mapState, mapStores } from 'pinia'

export default {
  computed: {
    ...mapState(useCounterStore, ['count', 'doubleCount']),
    ...mapStores(useCounterStore)
  }
}
```

---

## 四、State / Getters / Actions 详解

### State

- 定义：`state` 函数返回一个对象，所有属性就是状态
- 本质：使用 Vue 3 的 `reactive()` 包裹，整个对象是响应式的
- 访问：直接通过 `store.propertyName` 读写
- 修改：直接赋值 `store.count++` 或 `$patch`

### Getters

- 定义：函数接收 `state` 参数，返回派生值
- 本质：等价于 Vue 的 `computed`
- 缓存：有缓存，依赖不变时不重新计算
- 访问：`store.doubleCount`（不加括号，是属性不是方法）
- 访问其他 Getter：通过 `this`（Option Store）或直接调用（Setup Store）

### Actions

- 定义：普通函数（Option Store 中用 `this` 访问状态，Setup Store 中直接用 ref）
- 本质：同时替代了 Vuex 的 Mutations + Actions
- 支持异步：可以直接 `await`，无需像 Vuex 那样分两步
- 调用：`store.increment()` 直接调用

---

## 五、Pinia vs Vuex 对比表

| 维度 | Vuex | Pinia |
|------|------|-------|
| **Mutations** | 必须通过 Mutation 修改 State | ❌ 已移除，直接在 Actions 中修改 |
| **Modules** | 嵌套模块 + 命名空间 | ❌ 已移除，用多个独立 Store 代替 |
| **TypeScript** | 字符串 key，类型推导困难 | 泛型推导，完整类型支持 |
| **API 设计** | 4 个辅助函数（mapState 等） | 简化为 `storeToRefs` |
| **组合式 API** | 需要额外适配 | 原生支持，Store 本身就是组合函数 |
| **体积** | ~14KB | ~1.5KB |
| **根 Store** | 必须创建根 Store | 无根 Store，按需创建 |
| **DevTools** | 支持 | 完全支持，体验更好 |
| **SSR** | 支持 | 支持，设计更简洁 |
| **热更新** | 有限支持 | 原生支持 HMR |

---

## 六、Store 之间的调用

```js
// userStore.js
export const useUserStore = defineStore('user', {
  state: () => ({ name: 'Alice' })
})

// cartStore.js
import { useUserStore } from './userStore'

export const useCartStore = defineStore('cart', {
  actions: {
    checkout() {
      const user = useUserStore() // 在 Action 中直接调用其他 Store
      console.log(user.name)
    }
  }
})
```

---

## 七、回答问题

### Q1：Pinia 相比 Vuex 做了哪些改进？至少说3点。

1. **移除 Mutations**：Pinia 去掉了 Mutation 概念，State 可以直接在 Actions 中修改，减少样板代码，开发工具同样可以追踪状态变化
2. **移除 Modules 嵌套**：Vuex 的 Modules + 命名空间设计复杂，Pinia 用多个独立的 `defineStore` 代替，Store 之间可以直接互相调用，无需 `rootState` 和 `dispatch` 路径
3. **完整的 TypeScript 支持**：Vuex 的字符串 key 无法被 TS 推导，Pinia 通过泛型和函数式 API 实现了完整的类型推导
4. **原生 Composition API 支持**：Pinia 的 Setup Store 写法本质就是组合式函数，天然融入 Composition API 生态
5. **体积从 ~14KB 降到 ~1.5KB**：去掉 Mutations、Modules、辅助函数等冗余概念后，包体积大幅缩小

### Q2：Pinia 的核心 API 有哪些？怎么定义和使用一个 Store？

**核心 API：**
- `defineStore(id, options)` / `defineStore(id, setupFn)`：定义 Store
- `storeToRefs(store)`：解构 Store 保持响应式
- `store.$patch()`：批量修改状态
- `store.$reset()`：重置状态（仅 Option Store）
- `store.$subscribe()`：监听状态变化
- `store.$onAction()`：监听 Action 调用

**定义和使用流程：**
```js
// 1. 定义 Store（两种方式任选）
// Option Store
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: { double: (state) => state.count * 2 },
  actions: { increment() { this.count++ } }
})

// 2. 在组件中使用
import { useCounterStore } from './counterStore'
import { storeToRefs } from 'pinia'

const store = useCounterStore()            // 获取实例
const { count, double } = storeToRefs(store) // 解构保持响应式
const { increment } = store                 // actions 直接解构

// 3. 读写状态
console.log(store.count)   // 读
store.count++              // 直接修改
store.increment()          // 通过 action
store.$patch({ count: 5 }) // 批量修改
```

### Q3：Pinia 中的 State、Getters、Actions 分别对应什么？和 Vuex 的对应关系是什么？

| Pinia | Vuex | 说明 |
|-------|------|------|
| **State** | **State** | 直接对应，都是存储响应式状态，Pinia 用 `reactive()` 实现 |
| **Getters** | **Getters** | 直接对应，都是派生状态，本质都是 `computed`，有缓存 |
| **Actions** | **Mutations + Actions** | Pinia 的 Actions 同时承担了 Vuex 的 Mutation（同步修改状态）和 Action（异步操作）两个职责 |

**关键区别：**
- Vuex 中修改状态必须 `commit('mutation')`，异步操作必须 `dispatch('action')`，两步分离
- Pinia 中 Actions 既能同步修改也能异步操作，一步到位：
```js
// Vuex 的两步
actions: {
  async fetchData({ commit }) {
    const data = await api.getData()
    commit('SET_DATA', data)  // 必须通过 mutation
  }
}

// Pinia 的一步
actions: {
  async fetchData() {
    this.data = await api.getData()  // 直接修改，没有 mutation
  }
}
```
