---
type: review
---
# React Fiber 架构

## 概念梳理

### 核心概念一：React 15（Stack Reconciler）的问题

在 Fiber 之前，React 15 用的是 **Stack Reconciler**（栈协调器）。它的渲染过程是**同步的、不可中断的**。一旦开始渲染，就必须一口气把整个组件树递归完，中间不能停。

```
render App()
  → render Header()
    → render Nav()
      → render Logo()
  → render Content()
    → render List()
      → render Item() × 1000  ← 全部同步递归完成
  → render Footer()
```

**问题是什么？**

浏览器的一帧大约 **16.6ms**（60fps）。如果组件树很大，递归渲染超过了 16.6ms，就会**阻塞主线程**。主线程被占满了，用户的事件（点击、滚动、输入）就得不到响应，页面就**卡顿**了。

更严重的是，React 没有办法中断这个递归过程。一旦开始，必须全部完成。它不能说"我已经渲染了 5ms，先让浏览器处理一下用户的点击事件，再回来继续渲染"——**做不到**。

这就是 Fiber 要解决的核心问题：**让渲染过程可以中断、可以恢复。**

---

### 核心概念二：Fiber 是什么

Fiber 有两层含义：

**1. 一种新的协调算法（架构层面）**

React 16 引入了 Fiber Reconciler，替代了 Stack Reconciler。新的协调器可以把渲染工作拆分成**小单元**，每个单元执行完后可以检查"要不要暂停，让浏览器先处理别的事"。

**2. 一种数据结构（节点层面）**

每个 React 元素（组件、DOM 节点）都对应一个 Fiber 节点。Fiber 节点是 React 内部用来描述组件的**数据结构**，它替代了之前直接操作虚拟 DOM 树的方式。

#### Fiber 节点的数据结构

```javascript
const fiber = {
  // 静态结构信息
  tag: FunctionComponent,  // 组件类型（函数组件、类组件、DOM 等）
  type: App,               // 具体的函数/类/DOM 标签
  key: null,

  // 树结构 —— 三个指针构成 Fiber 树
  return: parentFiber,     // 指向父 Fiber 节点
  child: firstChildFiber,  // 指向第一个子 Fiber 节点
  sibling: nextFiber,      // 指向下一个兄弟 Fiber 节点

  // 工作单元
  pendingProps: {},        // 即将更新的 props
  memoizedProps: {},       // 上一次的 props
  memoizedState: {},       // 上一次的 state（Hooks 链表也挂在这里）
  updateQueue: null,       // 更新队列（setState 的更新存在这里）

  // 副作用
  flags: Placement,        // 需要执行的操作（插入、更新、删除等）
  alternate: currentFiber, // 指向另一棵树中对应的 Fiber（双缓冲机制）
}
```

#### Fiber 树的结构

Fiber 树不是用数组或对象树，而是通过 `child`、`sibling`、`return` 三个指针形成的一棵**链表树**：

```
App Fiber (return: null)
  │
  child → Header Fiber (return: App)
  │          │
  │          child → Nav Fiber (return: Header)
  │                     │
  │                     child → Logo Fiber (return: Nav)
  │
  sibling → Content Fiber (return: App)
  │             │
  │             child → List Fiber (return: Content)
  │
  sibling → Footer Fiber (return: App)
```

遍历顺序：
```
1. 从当前节点开始，有 child 就往下走（深度优先）
2. 没有 child 了，有 sibling 就走 sibling
3. 没有 sibling 了，走 return（回到父节点）
4. 父节点再找它的 sibling
5. 直到回到根节点，遍历完成
```

---

### 核心概念三：双缓冲机制（Double Buffering）

React 维护**两棵 Fiber 树**：

```
current 树：当前屏幕上显示的内容对应的 Fiber 树
workInProgress 树：正在构建中的新 Fiber 树
```

**工作流程：**
```
1. 基于 current 树，创建或复用 workInProgress 树的节点
2. 在 workInProgress 树上执行 Diff 和更新计算
3. workInProgress 树构建完成后，切换指针：
   current = workInProgress
   workInProgress 变成新的 current
4. 进入 Commit 阶段，把变更同步到真实 DOM
```

**为什么要两棵树？**
- 可以随时中断 workInProgress 树的构建（因为 current 树还在，屏幕显示不变）
- 中断后恢复时，可以继续基于 workInProgress 树继续工作
- 构建完成后一次性切换，保证 UI 变化是原子性的

每个 Fiber 节点的 `alternate` 指针就是指向另一棵树中对应的节点：
```
current App Fiber ←alternate→ workInProgress App Fiber
```

---

### 核心概念四：时间切片（Time Slicing）

#### 原理

Fiber 把渲染工作拆分成很多小的**工作单元（Work Unit）**，每个 Fiber 节点就是一个工作单元。每次执行一个工作单元后，React 会检查**是否已经用完了分配的时间片**（通常是 5ms）：

```javascript
while (workInProgress && !shouldYield()) {
  performUnitOfWork(workInProgress)  // 处理一个 Fiber 节点
}

// shouldYield() 检查：
// 从上一次让出主线程到现在，是否超过了时间片（5ms）
// 或者浏览器是否有更高优先级的任务要处理
```

**如果时间到了：**
- 保存当前的进度（workInProgress 树的当前节点已经记录了）
- 把主线程**交还给浏览器**
- 浏览器处理用户的点击、滚动、动画等
- 下一次空闲时（通过 `requestIdleCallback` 或 `MessageChannel` 调度），**从上次中断的地方继续**

这就是**可中断渲染**。用户不会感到页面卡住，因为 React 会不断让出主线程。

#### 不同优先级的调度

React 18 的并发特性中，更新有**优先级**：

```
紧急更新（同步）：用户输入、点击
高优先级：动画、交互反馈
普通优先级：网络请求回来的数据更新
低优先级：不紧急的分析计算
```

高优先级的更新可以**中断**低优先级的更新。比如用户正在输入，突然有网络数据返回，React 会先处理输入（高优先级），网络数据更新（低优先级）被中断，等输入处理完再继续。

---

### 核心概念五：Fiber 的两个阶段

React 的渲染分两个阶段，在 Fiber 架构下这两个阶段的行为不同：

#### Render 阶段（可中断）
- 遍历 Fiber 树，执行组件函数，生成新的 workInProgress 树
- 计算 Diff（哪些节点需要增删改）
- **可以被中断、恢复、丢弃**
- 纯 JS 计算，不操作 DOM

#### Commit 阶段（不可中断）
- 把 workInProgress 树的变更**同步应用到真实 DOM**
- 执行 useEffect / useLayoutEffect 的回调
- **不能中断**，必须一口气完成
- 一旦开始就不能停，否则 DOM 会处于不一致的状态

```
[Render 阶段（可中断）] → [Commit 阶段（不可中断）]
  可以被打断                    必须一口气完成
  不操作 DOM                    操作真实 DOM
  纯计算                        有副作用
```

这就是为什么时间切片只发生在 Render 阶段。Commit 阶段通常很快（只是操作 DOM），不需要中断。

---

### 核心概念六：Fiber 和 Hooks 的关系

Hooks 的实现依赖 Fiber 架构。每个 Fiber 节点的 `memoizedState` 上挂着一个**单向链表**，存储这个组件中所有的 Hooks。

```javascript
fiber.memoizedState = {
  memoizedState: count,  // useState 的 state
  queue: updateQueue,     // useState 的更新队列
  next: {                 // 下一个 Hook
    memoizedState: name,
    queue: null,
    next: {               // 下一个 Hook
      memoizedState: null,
      queue: effectQueue, // useEffect 的 effect
      next: null
    }
  }
}
```

这就是为什么 **Hooks 不能写在 if/for 里面**——React 依靠调用顺序来匹配每次 render 的 Hook 状态。如果顺序变了，链表就对不上了。

---

## 问题 1 — React 为什么要引入 Fiber？

**Stack Reconciler 的问题**：
- 递归遍历组件树，一旦开始就无法中断
- 大组件树会**长时间占用主线程**，阻塞用户输入、动画、布局
- 无法区分更新优先级，所有更新一视同仁

**Fiber 解决的问题**：
1. **可中断渲染**：把递归拆成链表遍历的循环，每次处理一个节点后可以暂停
2. **时间切片**：把长任务拆成小块，利用浏览器空闲时间执行，不阻塞 60fps
3. **优先级调度**：高优先级更新（用户输入）可以打断低优先级更新（数据预加载）

**一句话**：Fiber 让 React 从"一口气干完"变成了"干一点歇一点，重要的先干"。

---

## 问题 2 — Fiber 的数据结构与遍历

Fiber 节点是 React 内部描述组件的数据结构，通过三个指针（`child`、`sibling`、`return`）形成链表树，替代递归实现可中断的深度优先遍历。

```javascript
FiberNode {
  // --- 身份标识 ---
  tag: number,          // 组件类型（FunctionComponent / ClassComponent / HostComponent 等）
  type: any,            // 对应的函数/类/DOM 标签名
  key: null | string,   // React key

  // --- 链表指针（核心！）---
  child: Fiber | null,    // 第一个子节点
  sibling: Fiber | null,  // 下一个兄弟节点
  return: Fiber | null,   // 父节点

  // --- 状态 ---
  pendingProps: any,      // 新 props
  memoizedProps: any,     // 上次渲染的 props
  memoizedState: any,     // 上次渲染的 state（Hooks 链表也挂在这）
  updateQueue: any,       // 更新队列（setState / useState 的更新存在这里）

  // --- 副作用 ---
  flags: number,          // 标记（Placement / Update / Deletion 等）
  nextEffect: Fiber | null,

  // --- 双缓存 ---
  alternate: Fiber | null, // 对应另一棵树的 Fiber 节点

  // --- DOM ---
  stateNode: any,         // 真实 DOM 节点（HostComponent）或组件实例（ClassComponent）
}
```

**三个指针的遍历规则**：
- `child`：有子节点就往下走（深度优先）
- `sibling`：没有子节点了就走兄弟
- `return`：没有兄弟了就回到父节点

**遍历伪代码**：
```javascript
function workLoop(root) {
  let current = root.child
  while (current !== null) {
    // 1. 处理当前节点
    performWork(current)

    // 2. 有 child 就深入
    if (current.child) {
      current = current.child
      continue
    }

    // 3. 没 child 就找 sibling
    while (current.sibling === null && current.return !== null) {
      current = current.return  // return 回父节点
    }

    // 4. 走 sibling（或到根节点结束）
    current = current.sibling
  }
}
```

---

## 问题 3 — 时间切片（Time Slicing）

Fiber 把渲染工作拆分为以**单个 Fiber 节点**为单位的小任务，每处理完一个节点就检查是否该让出主线程。

**工作流程**：
```
[浏览器一帧 16.6ms]
├── 处理用户事件
├── 执行 JS（React 在这里工作）
├── 样式计算
├── 布局
└── 绘制

React 时间切片：
[JS 执行] ──5ms──→ 让出主线程 → [浏览器空闲] → [React 继续] → ...
```

**核心代码**：
```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress)
  }
}

// shouldYield() 检查：
// 从上一次让出主线程到现在，是否超过了时间片（5ms）
// 或者浏览器是否有更高优先级的任务要处理
```

**调度器（Scheduler）**：
- React 18 使用 **`MessageChannel`** 实现调度（优先），降级用 `requestIdleCallback`
- `MessageChannel` 在下一个宏任务中执行，比 `setTimeout` 更精确
- 调度器维护一个**优先级队列**，高优先级任务先执行

**为什么不用 setTimeout/setInterval？**
- 最小延迟 4ms，精度不够
- 嵌套层级多了最小延迟变成 4ms+
- 浏览器对它们有节流优化，不可靠

**优先级调度**：高优先级更新可以**中断**低优先级更新。比如用户正在输入时网络数据返回，React 先处理输入（高优先级），网络数据更新（低优先级）被中断，等输入处理完再继续。

**一句话总结**：时间切片就是"干一点活，看看有没有更重要的事，有就让路，没有就继续干"。
