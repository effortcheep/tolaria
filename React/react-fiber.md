---
type: review
---
# React Fiber 架构

## 概念梳理

### React 渲染机制的演进

**Stack Reconciler（React 15 及之前）**：
- 递归遍历虚拟 DOM 树，对比新旧 VDOM，找出差异
- 递归一旦开始就**无法中断**，必须遍历整棵树
- 组件树越大，主线程占用越久 → 页面卡顿、掉帧
- 浏览器每 **16.6ms** 需要渲染一帧（60fps），JS 执行超过这个时间用户就会感到卡

**Fiber Reconciler（React 16+）**：
- 把递归改成**循环**，每次只处理一个 Fiber 节点
- 可以在任意时刻**暂停、恢复、丢弃**
- 利用浏览器空闲时间执行，不阻塞用户交互和动画

### 核心概念

**Fiber 节点**：每个 React 元素对应一个 Fiber 节点，通过链表（child/sibling/return）连接成 Fiber 树。

**双缓存（Double Buffering）**：维护两棵树 — current（屏幕上的）和 workInProgress（正在构建的），commit 完成后 swap。

**优先级调度**：不同更新有不同优先级（用户交互 > 动画 > 数据更新），低优先级可被高优先级打断。

**时间切片**：拆分工作为小单元，每个单元执行后检查剩余时间，没时间就让出主线程。

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

## 问题 2 — Fiber 的数据结构

```js
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
  memoizedState: any,     // 上次渲染的 state
  updateQueue: any,       // 更新队列

  // --- 副作用 ---
  flags: number,          // 标记（插入/更新/删除）
  nextEffect: Fiber | null,

  // --- 双缓存 ---
  alternate: Fiber | null, // 对应另一棵树的 Fiber 节点

  // --- DOM ---
  stateNode: any,         // 真实 DOM 节点（HostComponent）或组件实例
}
```

**链表遍历逻辑**（替代递归）：
```
         App
        /   \
      Div    Span
      / \
    H1   P

→ child:  进入子节点
→ sibling: 进入兄弟节点
→ return:  回到父节点
```

**双缓存机制**：
- **current 树**：屏幕上的 UI
- **workInProgress 树**：正在构建的新树
- 渲染完成后，`root.current = workInProgress`，两棵树交换角色

---

## 问题 3 — 时间切片（Time Slicing）

**实现原理**：

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

**工作流程**：
1. 渲染工作被拆成以**单个 Fiber 节点**为单位的小任务
2. 每处理完一个 Fiber，就检查 `shouldYield()`（是否该让出）
3. 有剩余时间 → 继续处理下一个 Fiber
4. 没时间了 → **暂停**，让出主线程给浏览器处理高优任务
5. 浏览器空闲后 → **恢复**，从上次暂停的 Fiber 继续

**调度器（Scheduler）**：
- 使用 **`MessageChannel`**（优先）或 `requestIdleCallback` 实现
- `MessageChannel` 在下一个宏任务中执行，比 `setTimeout` 更精确
- 调度器维护一个**优先级队列**，高优先级任务先执行

**为什么不用 setTimeout/setInterval？**
- 最小延迟 4ms，精度不够
- 嵌套层级多了最小延迟变成 4ms+
- 浏览器对它们有节流优化，不可靠

**一句话总结**：时间切片就是"干一点活，看看有没有更重要的事，有就让路，没有就继续干"。
