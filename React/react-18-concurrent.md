---
type: review
---
# React 18 并发特性

## 概念梳理

### 核心概念一：并发模式（Concurrent Mode）的本质

React 18 引入的**并发（Concurrent）**不是一种功能，而是一种**新的底层机制**。

在 React 17 及之前，渲染是**同步的、不可中断的**——一旦 React 开始渲染更新，就必须一口气完成，中间不能停。这意味着如果有一个耗时的渲染任务，用户的输入、点击、动画都会被阻塞。

React 18 的并发模式让渲染变成了**可中断的、可调度的**：
- React 可以**同时准备多个版本的 UI**（不同状态的渲染结果）
- 高优先级更新（用户输入）可以**打断**低优先级更新（数据渲染）
- 浏览器不会因为渲染任务被阻塞，用户交互始终流畅

**关键区分**：
```
React 16 Fiber → 实现了可中断的架构基础（时间切片）
React 18 Concurrent → 在 Fiber 之上实现了优先级调度和并发渲染
```

Fiber 解决的是"能不能中断"的问题，Concurrent 解决的是"中断后怎么调度"的问题。

---

### 核心概念二：过渡（Transition）—— 更新的优先级分类

React 18 把更新分成了两类：

**紧急更新（Urgent Update）**：
- 用户直接触发的交互：打字、点击、滚动
- 需要**立即响应**，延迟用户就能感知到卡顿
- 示例：输入框的 `value` 变化、按钮的点击反馈

**过渡更新（Transition Update）**：
- 由紧急更新**间接导致**的 UI 变化：搜索结果列表、Tab 切换的内容、页面导航
- 可以**延迟一点**，用户能接受短暂的加载状态
- 示例：输入搜索词后，搜索结果列表的更新

```
用户输入 "react"（紧急更新，立即响应）
  → 搜索结果列表重新渲染（过渡更新，可以慢一点）
  → 显示 loading 状态，而不是卡住输入框
```

**Transition 的核心价值**：让 React 知道哪些更新是"可以等一等"的，从而把主线程让给更紧急的任务。

---

### 核心概念三：Suspense —— 声明式的异步 UI 状态

`Suspense` 是 React 提供的一种**声明式**处理异步状态的方式。

**传统做法（命令式）**：
```jsx
function SearchResults() {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)

  useEffect(() => {
    setLoading(true)
    fetchData()
      .then(data => setData(data))
      .catch(err => setError(err))
      .finally(() => setLoading(false))
  }, [])

  if (loading) return <Spinner />
  if (error) return <ErrorPage />
  return <List data={data} />
}
```

**Suspense 做法（声明式）**：
```jsx
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <SearchResults />
    </Suspense>
  )
}
```

组件内部不需要写 loading/error 状态管理，直接"假设数据已经存在"去渲染。如果数据还没准备好，React 会自动显示 `fallback`。

**工作原理**：
- `Suspense` 会捕获其子组件树中**任何处于 pending 状态的异步操作**
- 当子组件"挂起"（suspended）时，显示 `fallback`
- 当异步操作完成，自动切换到真实 UI
- 一个 `Suspense` 可以包裹多个子组件，只要**所有子组件都就绪**才显示真实 UI

**边界规则**：
```
<Suspense fallback={<HeaderSkeleton />}>    ← Header 专用边界
  <Header />
</Suspense>
<Suspense fallback={<ContentSkeleton />}>   ← Content 专用边界
  <Content />
  <Sidebar />                               ← 两个都在这个边界内
</Suspense>
```

---

### 核心概念四：createRoot vs ReactDOM.render —— 并发渲染的入口

React 18 的并发特性**不是自动开启的**，需要通过 `createRoot` API 显式启用。

**React 17 的方式**：
```jsx
import ReactDOM from 'react-dom'
ReactDOM.render(<App />, document.getElementById('root'))
```
- 使用的是**同步渲染器**
- 所有更新都是同步的、不可中断的
- 即使你用了 React 18 的 API（如 `useTransition`），也不会生效

**React 18 的方式**：
```jsx
import { createRoot } from 'react-dom/client'
const root = createRoot(document.getElementById('root'))
root.render(<App />)
```
- 使用的是**并发渲染器**
- 更新默认就是可中断的、可调度的
- `useTransition`、`useDeferredValue`、`Suspense` 的并发能力才能真正生效

**为什么要换 API？**
- 这是一个**显式的 opt-in** 选择，避免老项目升级后出现意外行为
- 并发模式可能改变某些副作用的执行时机，需要开发者确认自己的代码兼容
- React 团队的做法：新项目用 `createRoot`，老项目可以逐步迁移

---

### 核心概念五：自动批处理（Automatic Batching）

**React 17 的批处理**：
- 只在 React 事件处理函数中自动批处理（多次 `setState` 合并为一次渲染）
- 在 `setTimeout`、`Promise`、原生事件中**不会批处理**，每次 `setState` 都触发一次渲染

```jsx
// React 17 中
function handleClick() {
  // ✅ 在 React 事件中，批处理 → 只渲染 1 次
  setCount(c => c + 1)
  setFlag(f => !f)
}

setTimeout(() => {
  // ❌ 在 setTimeout 中，不批处理 → 渲染 2 次
  setCount(c => c + 1)
  setFlag(f => !f)
}, 0)
```

**React 18 的自动批处理**：
- **所有更新默认都自动批处理**，不管在哪里触发
- `setTimeout`、`Promise`、原生事件、`await` 之后的更新，全部合并

```jsx
// React 18 中
setTimeout(() => {
  // ✅ 现在也批处理了 → 只渲染 1 次
  setCount(c => c + 1)
  setFlag(f => !f)
}, 0)

fetch('/api').then(() => {
  // ✅ Promise 中也批处理了
  setData(result)
  setLoading(false)
})
```

**为什么这是并发模式的基础？**
- 批处理减少了不必要的渲染次数
- 渲染次数少了，时间切片的调度就更灵活
- 如果每次 `setState` 都渲染，就没办法做优先级调度了

---

### 核心概念六：useTransition 与 useDeferredValue 的定位

这两个 API 都是用来处理**非紧急更新**的，但使用场景不同：

**useTransition** —— "我有一个状态更新，想把它标记为过渡"
- 你**控制**状态更新的时机
- 在事件处理函数中，把某个 `setState` 包在 `startTransition` 里
- React 知道这个更新是低优先级的，可以被中断

**useDeferredValue** —— "我有一个已经存在的值，想让它的更新慢一点"
- 你**不控制**状态更新（可能是父组件传下来的 props）
- 对一个**已有的值**做延迟处理
- React 会先用旧值渲染，等空闲了再用新值渲染

**两者本质相同**：都是告诉 React "这个更新不紧急，可以被中断"。区别在于控制权：
```
useTransition → 你主动标记某个 setState 是过渡
useDeferredValue → 你被动地让某个值的更新变慢
```

---

## 问题 1 — useTransition 和 useDeferredValue 分别做什么？有什么区别？

### useTransition

`useTransition` 返回一个数组：`[isPending, startTransition]`。

- `startTransition(callback)` — 把 callback 里的 `setState` 标记为**过渡更新**
- `isPending` — 布尔值，表示过渡更新是否还在进行中（可以用来显示 loading）

```jsx
function TabContainer() {
  const [tab, setTab] = useState('home')
  const [isPending, startTransition] = useTransition()

  function selectTab(nextTab) {
    // 紧急更新：立即切换高亮状态
    setTab(nextTab)
    // 过渡更新：Tab 内容可以慢一点渲染
    startTransition(() => {
      setTab(nextTab)  // 实际上这里只保留一个 setTab，用 startTransition 包裹
    })
  }

  return (
    <div>
      <TabBar onChange={selectTab} activeTab={tab} />
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </div>
  )
}
```

**实际用法**：
```jsx
function SearchPage() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [isPending, startTransition] = useTransition()

  function handleChange(e) {
    // 紧急更新：输入框立即响应
    setQuery(e.target.value)
    // 过渡更新：搜索结果可以延迟
    startTransition(() => {
      setResults(filterData(e.target.value))
    })
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <span>搜索中...</span>}
      <ResultList results={results} />
    </div>
  )
}
```

### useDeferredValue

`useDeferredValue(value)` 接收一个值，返回一个**延迟版本**的值。

```jsx
function SearchPage({ query }) {
  // query 是父组件传来的，你控制不了它的更新时机
  const deferredQuery = useDeferredValue(query)

  return (
    <div>
      <input value={query} />
      {/* deferredQuery 会"落后"于 query，等空闲了才更新 */}
      <ResultList query={deferredQuery} />
    </div>
  )
}
```

**工作原理**：
```
用户快速输入 "r" → "re" → "rea" → "reac" → "react"

query:          r → re → rea → reac → react    （紧急，立即响应）
deferredQuery:  r → re → rea → reac → react    （延迟，可能跳过中间值）

React 先用旧的 deferredQuery 渲染结果列表
等输入停下来（或浏览器空闲），再用最新的 deferredQuery 重新渲染
```

### 两者的区别

| 维度 | useTransition | useDeferredValue |
|---|---|---|
| **控制对象** | 控制**状态更新**（setState） | 控制**已有值**的延迟 |
| **使用场景** | 你触发的更新，能包在 startTransition 里 | 值来自 props 或其他地方，你控制不了更新 |
| **返回值** | `[isPending, startTransition]` | 延迟版本的值 |
| **loading 状态** | 自带 `isPending` | 需要自己用 `deferredValue !== value` 判断 |
| **本质** | 把更新标记为过渡 | 对值做延迟副本 |

**什么时候用哪个？**
- 你能控制 `setState` 的调用 → 用 `useTransition`
- 值是 props 传进来的，你控制不了更新时机 → 用 `useDeferredValue`

**一句话总结**：`useTransition` 是"我主动把更新降级"，`useDeferredValue` 是"我被动让值慢一点"。

---

## 问题 2 — Suspense 的作用是什么？

### 核心作用

`Suspense` 让组件可以**"等待"异步操作完成**，在等待期间显示 fallback UI，完成后自动切换到真实 UI。

```jsx
<Suspense fallback={<Loading />}>
  <SomeAsyncComponent />
</Suspense>
```

### 三个核心能力

**1. 声明式加载状态**

不再需要手动管理 `loading` / `error` / `data` 三态，直接声明"这个区域在数据准备好之前显示什么"。

```jsx
// ❌ 命令式：每个组件自己管理 loading
function ProfilePage() {
  const [user, setUser] = useState(null)
  const [posts, setPosts] = useState(null)
  useEffect(() => { fetchUser().then(setUser) }, [])
  useEffect(() => fetchPosts().then(setPosts), [])

  if (!user || !posts) return <Spinner />
  return <div>{user.name}<PostList posts={posts} /></div>
}

// ✅ 声明式：Suspense 统一管理
function ProfilePage() {
  return (
    <Suspense fallback={<Spinner />}>
      <ProfileData />
    </Suspense>
  )
}

function ProfileData() {
  // 直接"假设数据存在"去渲染，如果没准备好会自动 suspend
  const user = use(fetchUser())
  const posts = use(fetchPosts())
  return <div>{user.name}<PostList posts={posts} /></div>
}
```

**2. 细粒度的加载边界**

可以为不同的 UI 区域设置不同的 `Suspense` 边界，实现**渐进式加载**——用户先看到页面骨架，再逐步看到各个区域的内容。

```jsx
function App() {
  return (
    <Layout>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />          ← Header 先加载完就先显示
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <Content />         ← Content 加载完再显示
        <Sidebar />
      </Suspense>
    </Layout>
  )
}
```

**3. 配合并发特性**

在 React 18 中，`Suspense` 配合 `useTransition` 可以实现**非阻塞的加载状态**：

```jsx
function SearchPage() {
  const [query, setQuery] = useState('')
  const [isPending, startTransition] = useTransition()

  function handleChange(e) {
    setQuery(e.target.value)  // 紧急：输入框立即响应
    startTransition(() => {
      // 过渡：触发 Suspense 边界，但不会阻塞输入
      setSearchQuery(e.target.value)
    })
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Suspense fallback={<ResultSkeleton />}>
        <SearchResults />
      </Suspense>
    </div>
  )
}
```

### 一句话总结

`Suspense` 的作用是**用声明式的方式处理异步 UI 的加载状态**——组件不需要自己管理 loading 逻辑，只需要"假设数据存在"去渲染，`Suspense` 负责处理"数据还没准备好"的情况。

---

## 问题 3 — React 18 的 createRoot 和之前的 ReactDOM.render 有什么区别？

### 本质区别

| 维度 | `ReactDOM.render` (React 17) | `createRoot` (React 18) |
|---|---|---|
| **渲染器** | 同步渲染器 | 并发渲染器 |
| **渲染方式** | 同步、不可中断 | 可中断、可调度 |
| **批处理** | 只在 React 事件中批处理 | 所有场景自动批处理 |
| **并发 API** | 不生效 | 生效（useTransition 等） |
| **SSR hydration** | `ReactDOM.hydrate` | `hydrateRoot` |
| **严格模式** | 不严格检查 | 开发模式下会渲染两次以检测副作用 |

### 代码对比

```jsx
// ❌ React 17
import ReactDOM from 'react-dom'
ReactDOM.render(<App />, document.getElementById('root'))

// ✅ React 18
import { createRoot } from 'react-dom/client'
const root = createRoot(document.getElementById('root'))
root.render(<App />)
```

### 为什么不直接升级 ReactDOM.render？

**显式 opt-in 设计**：
- 并发模式会改变某些副作用的执行时机（比如 `useEffect` 可能执行更晚）
- 老代码可能依赖"渲染是同步的"这个假设
- 直接升级可能导致 bug，所以需要开发者**主动切换**到 `createRoot`

**React 团队的迁移策略**：
```jsx
// 第一步：升级到 React 18，继续用 ReactDOM.render（兼容模式）
// 第二步：把 ReactDOM.render 换成 createRoot（启用并发模式）
// 第三步：逐步使用 useTransition 等新 API 优化体验
```

### createRoot 的其他变化

**1. 不再隐式复用 container**
```jsx
// React 17：多次调用 render 会复用 container
ReactDOM.render(<App1 />, container)
ReactDOM.render(<App2 />, container)  // 直接替换

// React 18：需要显式调用 root.render
const root = createRoot(container)
root.render(<App1 />)
root.render(<App2 />)  // 同一个 root 上替换
```

**2. 卸载方式变化**
```jsx
// React 17
ReactDOM.unmountComponentAtNode(container)

// React 18
root.unmount()
```

**3. 回调函数不再需要**
```jsx
// React 17：render 的第二个参数是回调
ReactDOM.render(<App />, container, () => {
  console.log('渲染完成')
})

// React 18：没有回调参数，用 useEffect 代替
```

### 一句话总结

`createRoot` 是 React 18 并发渲染的**入口开关**——用它才能启用自动批处理、时间切片、`useTransition`、`Suspense` 并发模式等所有新特性。`ReactDOM.render` 是旧的同步渲染入口，保留是为了兼容。
