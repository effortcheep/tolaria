---
type: review
---
# React Hooks

## 概念梳理

**Hooks 的本质**：让函数组件拥有状态和生命周期能力的函数，以 `use` 开头，遵循两条规则——只在顶层调用、只在函数组件或自定义 Hook 中调用。

**Hooks 分类**：
- **状态类**：`useState`、`useReducer` — 管理组件内部数据
- **副作用类**：`useEffect`、`useLayoutEffect` — 处理副作用
- **性能优化类**：`useMemo`、`useCallback` — 避免不必要的重新计算/渲染
- **引用类**：`useRef` — 持久化可变容器，不触发重渲染

**渲染机制核心**：`useState` 的 setter 触发重渲染，`useRef` 修改 `.current` 不触发；渲染时所有 Hook 按顺序执行，不能放在条件/循环里。

---

## 1. `useState` vs `useRef`

| 维度 | `useState` | `useRef` |
|---|---|---|
| **本质** | 状态管理 | 可变容器（"盒子"） |
| **修改方式** | `setState(newValue)` | `ref.current = newValue` |
| **是否触发重渲染** | ✅ 是 | ❌ 否 |
| **值的持久性** | 跨渲染保持 | 跨渲染保持 |
| **异步读取** | 不能立即读到最新值（批量更新） | 立即读到最新值 |

**典型场景**：

```jsx
// useState — 需要驱动 UI 更新
const [count, setCount] = useState(0)
// count 变了 → 组件重渲染 → 界面更新

// useRef — 保存不需要渲染到界面的值
const timerRef = useRef(null)
timerRef.current = setInterval(() => { /* ... */ })
// 修改 timerRef.current 不触发重渲染

const inputRef = useRef(null)
inputRef.current.focus()  // DOM 引用
```

**一句话总结**：要更新界面用 `useState`，不要更新界面用 `useRef`。

---

## 2. `useEffect` 依赖数组

```jsx
useEffect(() => { /* 回调 */ }, [依赖])
```

| 写法 | 行为 | 等价于 |
|---|---|---|
| `useEffect(fn)` | **每次渲染后**都执行 | 所有生命周期 |
| `useEffect(fn, [])` | **只在挂载后**执行一次 | componentDidMount |
| `useEffect(fn, [a, b])` | **挂载后 + a 或 b 变化时**执行 | componentDidMount + 部分 componentDidUpdate |

```jsx
// 1. 没有依赖数组 → 每次渲染都执行
useEffect(() => {
  console.log('每次渲染后都会执行')
})

// 2. 空数组 → 只执行一次（挂载时）
useEffect(() => {
  console.log('只在组件挂载时执行一次')
  return () => console.log('卸载时清理')
}, [])

// 3. 有依赖 → 依赖变化时执行
useEffect(() => {
  console.log(`count 变为 ${count}`)
}, [count])  // count 变了才重新执行
```

**清理函数**：`useEffect` 返回的函数在**下次 effect 执行前**或**组件卸载时**调用，用于清理定时器、取消订阅等。

---

## 3. `useMemo` vs `useCallback`

两者都是**性能优化手段**，缓存计算结果，避免每次渲染都重新执行。

| 维度 | `useMemo` | `useCallback` |
|---|---|---|
| **缓存什么** | **值**（计算结果） | **函数**（函数引用） |
| **返回值** | 计算结果 `value` | 函数本身 `fn` |
| **本质** | `useMemo(() => value, deps)` | `useMemo(() => fn, deps)` 的语法糖 |
| **使用场景** | 昂贵的计算、对象引用稳定 | 传给子组件的回调、作为其他 Hook 的依赖 |

```jsx
// useMemo — 缓存计算结果
const expensiveResult = useMemo(() => {
  return heavyCompute(data)  // 只在 data 变化时重新计算
}, [data])

// useCallback — 缓存函数引用
const handleClick = useCallback(() => {
  setCount(c => c + 1)  // 函数引用不变，子组件不会因父组件重渲染而重渲染
}, [])

// 以下两者完全等价：
useCallback(fn, deps)
useMemo(() => fn, deps)
```

**为什么要用**：
- 避免**不必要的子组件重渲染**（配合 `React.memo`）
- 避免**昂贵计算的重复执行**
- 避免**每次渲染创建新引用**导致下游 effect 误触发

### `useMemo` / `useCallback` 与 re-render 的关系

**核心误区**：`useMemo` / `useCallback` **不能阻止当前组件的 re-render**。父组件状态变了，父组件照样重渲染，它们缓存的值/函数引用没变而已。

**要阻止子组件不必要的 re-render，必须 `React.memo` + `useMemo` / `useCallback` 配合使用**：

```jsx
function Parent() {
  const [count, setCount] = useState(0)

  // ❌ count 变化时 Parent 照样 re-render
  // useMemo/useCallback 不能阻止当前组件的 re-render
  const expensiveValue = useMemo(() => computeSomething(), [])
  const handleClick = useCallback(() => {}, [])

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child value={expensiveValue} onClick={handleClick} />
    </div>
  )
}

// ✅ React.memo 检查 props 是否变化，没变就跳过 re-render
const Child = React.memo(function Child({ value, onClick }) {
  console.log('Child rendered')
  return <button onClick={onClick}>{value}</button>
})
```

**工作原理**：
1. `useCallback` 缓存函数引用 → `handleClick` 始终是同一个引用
2. `useMemo` 缓存值 → `expensiveValue` 引用不变
3. `React.memo` 浅比较 props → 发现 `value` 和 `onClick` 都没变 → **跳过 Child 的 re-render**

**一句话总结**：`useMemo` / `useCallback` 负责**稳定引用**，`React.memo` 负责**拦截 re-render**，两者缺一不可。

**一句话总结**：`useMemo` 缓存**值**，`useCallback` 缓存**函数**，本质都是 `useMemo` 的特化。
