---
type: review
---
# React 状态管理

## 一、核心概念

### 1. 状态的分类

| 分类 | 说明 | 示例 |
|------|------|------|
| **局部状态 (Local State)** | 单个组件内部管理的状态 | 表单输入、展开/折叠、hover 状态 |
| **提升状态 (Lifted State)** | 提升到最近公共父组件，通过 props 下发 | 兄弟组件共享的数据 |
| **全局状态 (Global State)** | 跨越组件树多层共享的状态 | 用户登录信息、主题、语言、购物车 |
| **服务端状态 (Server State)** | 来自远程 API 的数据，本质是缓存 | 用户列表、文章详情、搜索结果 |
| **URL 状态** | URL 中的参数，可序列化、可分享 | 搜索关键词、分页码、tab 选中项 |

> **原则**：优先使用局部状态，按需逐级提升。全局状态管理是最后手段，不是默认选择。

### 2. React 内置状态管理方案

| 方案 | 作用域 | 触发方式 | 适用场景 |
|------|--------|----------|----------|
| `useState` | 组件级 | 直接 setState | 简单局部状态 |
| `useReducer` | 组件级 | dispatch action | 复杂状态逻辑、多个子值相互依赖 |
| `useContext` | 跨组件树 | Context Provider | 低频变化的全局值（主题、语言、认证） |

---

## 二、useContext

### 作用

`useContext` 让组件**无需逐层传递 props**就能读取上层 Context 中的值。它解决了 **prop drilling（props 层层传递）** 的问题。

### 工作机制

```
Context.Provider (value={...})
    └── 子组件 A
         └── 子组件 B
              └── 孙组件 C ← useContext(MyContext) 直接拿到值
```

- `createContext` 创建 Context 对象
- `Provider` 在上层提供值
- 任意后代组件通过 `useContext(Context)` 消费值
- 当 Provider 的 `value` 变化时，**所有**消费该 Context 的组件都会重新渲染

### 局限

| 局限 | 说明 |
|------|------|
| **性能问题** | Context 值变化时，**所有**消费者组件无条件重渲染，无法精确订阅部分值 |
| **不适合高频更新** | 动画、实时数据、频繁交互等场景会导致大量无效渲染 |
| **不适合复杂状态逻辑** | 没有内置的中间件、devtools、时间旅行调试等能力 |
| **嵌套地狱** | 多个 Context 嵌套时 Provider 层级过深，代码可读性下降 |
| **无法在组件外读取** | 只能在组件渲染期间调用，无法在事件处理或异步代码中直接访问 |

### 最佳实践

- Context 适合存放**变化频率低**的全局值：主题、语言、认证状态、当前用户
- 对于高频更新或复杂状态，应使用专用状态管理库

---

## 三、Redux 核心概念

### 三大核心

#### Store（仓库）
- 整个应用的**单一数据源**（Single Source of Truth）
- 保存完整的状态树（state tree）
- 通过 `createStore(reducer)` / `configureStore()` 创建
- 提供 `getState()`、`dispatch(action)`、`subscribe(listener)` 三个方法

#### Action（动作）
- 描述"发生了什么"的**普通对象**
- 必须有 `type` 字段，可携带 `payload` 数据
```js
{ type: 'todos/add', payload: '学习 Redux' }
```
- **Action Creator**：创建 Action 的工厂函数
```js
const addTodo = (text) => ({ type: 'todos/add', payload: text })
```

#### Reducer（归约器）
- 纯函数：`(state, action) => newState`
- 根据 action.type 决定如何更新状态
- **必须不可变**：不能修改原 state，必须返回新对象
```js
function todoReducer(state = [], action) {
  switch (action.type) {
    case 'todos/add':
      return [...state, { text: action.payload, done: false }]
    case 'todos/toggle':
      return state.map((t, i) =>
        i === action.index ? { ...t, done: !t.done } : t
      )
    default:
      return state
  }
}
```

### 单向数据流

```
View → dispatch(Action) → Reducer(state, action) → newState → Store → View 更新
```

1. 组件触发 `dispatch(action)`
2. Store 调用 Reducer，传入当前 state 和 action
3. Reducer 返回新 state
4. Store 通知所有订阅者（`subscribe`）
5. 组件从 Store 读取新 state 并重新渲染

### Redux 三大原则

| 原则 | 说明 |
|------|------|
| **单一数据源** | 整个应用只有唯一一个 Store |
| **State 只读** | 唯一修改 state 的方式是 dispatch action |
| **纯函数执行** | Reducer 必须是纯函数，不能有副作用 |

### Redux 中间件

中间件是对 `dispatch` 的增强，形成洋葱模型：
```
dispatch → middleware 1 → middleware 2 → ... → reducer
```

常用中间件：
- **redux-thunk**：处理异步逻辑（dispatch 函数而非对象）
- **redux-saga**：使用 Generator 管理复杂异步流程
- **RTK Query / RTK Query**：RTK 内置的数据获取方案

---

## 四、Redux Toolkit (RTK)

RTK 是 Redux 官方推荐的**标准编写方式**，解决了原生 Redux 样板代码过多的问题。

| API | 作用 |
|-----|------|
| `configureStore` | 替代 `createStore`，自动集成 thunk、devtools |
| `createSlice` | 自动生成 action types + action creators + reducer |
| `createAsyncThunk` | 标准化异步请求的 pending/fulfilled/rejected 处理 |
| `createSelector` | Reselect 集成，创建记忆化 selector |

```js
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    add(state, action) {
      state.push({ text: action.payload, done: false }) // Immer 允许"直接修改"
    },
  },
})
// 自动生成 todosSlice.actions.add 和 todosSlice.reducer
```

> RTK 内部使用 **Immer**，允许在 reducer 中写"可变"语法，实际生成不可变的新 state。

---

## 五、现代状态管理库对比

### Zustand

| 特点 | 说明 |
|------|------|
| **API 极简** | 一个 `create` 函数搞定一切 |
| **无 Provider** | 不需要包裹 Provider 组件 |
| **选择性订阅** | 自动浅比较，只在选中值变化时重渲染 |
| **支持中间件** | persist、devtools、immer 等 |
| **体积** | ~1KB gzip |

```js
import { create } from 'zustand'

const useStore = create((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 })),
}))

// 组件中使用
function Counter() {
  const count = useStore((s) => s.count) // 只订阅 count
  return <button onClick={useStore.getState().inc}>{count}</button>
}
```

### Jotai

| 特点 | 说明 |
|------|------|
| **原子化** | 每个状态是独立的 atom，类似 Recoil 的思路 |
| **自底向上** | 先定义原子，再组合，而非先定义大 store |
| **自动依赖追踪** | derived atoms 自动追踪依赖并缓存 |
| **无 Provider** | 可选的 Provider，也可以无 Provider 使用 |
| **并发兼容** | 与 React 18 concurrent features 兼容良好 |

```js
import { atom, useAtom } from 'jotai'

const countAtom = atom(0)
const doubleAtom = atom((get) => get(countAtom) * 2) // derived atom

function Counter() {
  const [count, setCount] = useAtom(countAtom)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### 三者对比

| 维度 | Redux (RTK) | Zustand | Jotai |
|------|-------------|---------|-------|
| **理念** | 集中式、可预测、单向数据流 | 轻量 store，最小化 API | 原子化，自底向上 |
| **样板代码** | 较多（即使 RTK 简化了） | 极少 | 极少 |
| **学习曲线** | 较高（概念多） | 低 | 低 |
| **Provider** | 必须 | 不需要 | 可选 |
| **DevTools** | Redux DevTools（功能最强） | 支持 | 支持 |
| **中间件** | 生态最丰富 | persist/devtools/immer | jotai/utils |
| **异步处理** | thunks / sagas / RTK Query | 自行处理 | 异步 atom |
| **TypeScript** | 好（RTK 原生 TS） | 好 | 好 |
| **适用规模** | 中大型、团队协作 | 中小型 | 中小型 |
| **包体积** | ~11KB (RTK) | ~1KB | ~2KB |

---

## 六、为什么现在很多人不用 Redux 了？

### 核心原因

1. **过度设计**：很多应用的全局状态需求很简单（主题、用户、语言），不需要 Redux 的整套架构
2. **样板代码过多**：即使 RTK 简化了，action types / actions / reducers / selectors 的心智模型仍比 Zustand/Jotai 复杂
3. **Context + useReducer 够用**：对于中小型应用，内置方案 + 适度性能优化已能满足需求
4. **React Server Components**：服务端组件可以直接获取数据，减少了客户端全局状态的需求
5. **服务端状态 ≠ 客户端状态**：TanStack Query / SWR 等库将服务端数据作为缓存管理，Redux 在这个场景下并非最佳选择
6. **原子化模型更灵活**：Jotai/Recoil 的 atom 模型天然支持按需订阅，避免了 Context 的全局重渲染问题

### Redux 仍然有价值的场景

- 大型团队需要统一的状态管理规范和中间件生态
- 需要时间旅行调试、action 日志追踪
- 业务逻辑复杂，需要 sagas 或 RTK Query 管理异步流程
- 已有大量 Redux 代码的存量项目

---

## 七、回答问题

### Q1: useContext 的作用是什么？它解决了什么问题？有什么局限？

**作用**：`useContext` 让组件无需逐层传递 props 就能读取上层 Context 中的值。

**解决的问题**：**Prop Drilling** — 当一个值需要从祖先组件传递到深层后代时，中间每一层都要接收并转发 props，即使它们并不使用这个值。useContext 让深层组件直接"穿透"中间层获取数据。

**局限**：
- **性能**：Context 值变化时所有消费者无条件重渲染，无法精确订阅
- **不适合高频更新**：频繁变化的状态（动画、实时数据）会导致大量无效渲染
- **无复杂状态管理能力**：没有中间件、devtools、时间旅行调试等
- **嵌套地狱**：多个 Context 叠加导致 Provider 层级过深

### Q2: Redux 的核心概念（Store、Action、Reducer）分别是什么？数据流向是怎样的？

**Store**：应用的单一数据源，保存完整的状态树。提供 `getState()`、`dispatch()`、`subscribe()` 三个方法。

**Action**：描述"发生了什么"的普通对象，必须有 `type` 字段，可携带 `payload`。

**Reducer**：纯函数 `(state, action) => newState`，根据 action type 计算新状态，不修改原 state。

**数据流向**（单向数据流）：
```
View → dispatch(Action) → Reducer → newState → Store → View 更新
```
组件调用 `dispatch(action)` → Store 将当前 state 和 action 传给 Reducer → Reducer 返回新 state → Store 更新并通知订阅的组件 → 组件重新渲染。

### Q3: Redux 和 Zustand、Jotai 相比有什么区别？为什么现在很多人不用 Redux 了？

**区别**：
- **Redux**：集中式 store，强调单向数据流和可预测性，概念多（store/action/reducer/middleware），样板代码较多
- **Zustand**：极简 API，一个 `create` 函数，无 Provider，支持选择性订阅，~1KB
- **Jotai**：原子化模型，每个状态是独立 atom，自底向上组合，自动依赖追踪

**不用 Redux 的原因**：
1. 很多应用不需要复杂的集中式状态管理
2. Zustand/Jotai 的 API 更简单、学习成本更低
3. React Server Components 减少了客户端全局状态的需求
4. TanStack Query 等库将服务端数据从客户端状态中分离出来
5. Context + useReducer 对中小型应用已够用

Redux 在大型团队、复杂业务、需要强大中间件生态的场景中仍有价值。
