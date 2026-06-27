---
type: review
relationships:
  Type:
    - "[[review]]"
---
# 垃圾回收（Garbage Collection）

## 核心概念：为什么需要垃圾回收？

### 栈内存 vs 堆内存

JavaScript 引擎将内存分为两个区域：

| | 栈内存（Stack） | 堆内存（Heap） |
|---|---|---|
| **存什么** | 原始类型值、函数调用帧、引用地址 | 对象、数组、函数等引用类型数据 |
| **分配方式** | 自动（进/出调用栈时分配/释放） | 手动申请，由 GC 自动回收 |
| **大小** | 小（通常几 MB） | 大（可达数 GB） |
| **管理** | 不需要 GC | **GC 的管理对象** |

```
执行代码时：

  栈（Stack）                    堆（Heap）
  ┌──────────┐                  ┌───────────────┐
  │ fn() 调用帧 │                 │               │
  │  a: ──────────────────────→ │ { x: 1, y: 2 }│
  │  b: 3      │                └───────────────┘
  ├──────────┤
  │ 全局作用域  │
  │  obj: ────────────────────→ ┌───────────────┐
  │            │                │ [1, 2, 3, 4]  │
  └──────────┘                └───────────────┘

fn() 返回后，栈帧弹出，a 消失，
但堆上的对象可能仍被其他引用持有 → 需要 GC 判断
```

**关键点**：原始类型（number、string、boolean 等）存在栈上，函数返回即自动释放；引用类型（对象、数组等）存在堆上，需要 GC 来判断何时释放。

### 可达性（Reachability）— GC 的判断依据

GC 的核心问题：**堆上哪些对象仍然有用？**

答案是**可达性**。从根对象（GC Root）出发，沿着引用链能访问到的对象就是「可达的」（有用的），访问不到的就是垃圾。

**GC Root 包括：**
- 全局对象（`window` / `globalThis` / `global`）
- 当前调用栈上的所有局部变量和参数
- 闭包中捕获的变量
- DOM 树中所有节点（被 JS 引用的）
- 活跃的定时器、Promise 回调

```
GC Root（根对象）
  │
  ├── window.document → DOM 树...
  │
  ├── 当前函数作用域
  │     └── myObj ──→ { data: [...] }
  │
  └── 闭包捕获
        └── cache ──→ Map { ... }
                            │
                            └──→ 大数组（可达，不能回收）

不可达的对象（没有任何引用链能从 Root 到达）→ 垃圾 → 回收
```

### JavaScript 的自动内存管理

JavaScript 使用**自动内存管理**——开发者创建对象、使用对象，引擎自动判断哪些对象不再需要并释放其内存。这个过程叫垃圾回收（GC）。

---

## 1. JavaScript 的垃圾回收机制是什么？有哪几种主流算法？

### 两种主流算法

#### ① 标记-清除（Mark-Sweep）— 最核心、最常用

- **标记阶段**：从根对象出发，递归遍历所有引用链，标记所有可达对象
- **清除阶段**：遍历堆内存，回收所有未被标记的对象

```
标记前：  [A] → [B] → [C]    [D] → [E]    [F]
                      ↑ 根可达         ↑ 不可达

标记后：  [A✓] → [B✓] → [C✓]    [D✗] → [E✗]    [F✗]

清除后：  [A] → [B] → [C]    （空闲内存）   （空闲内存）
```

**优点**：实现简单，天然处理循环引用（因为判断依据是可达性，不是引用计数）
**缺点**：产生**内存碎片**（回收后空间不连续）

#### ② 标记-整理（Mark-Compact）— 解决碎片问题

- 标记阶段与 Mark-Sweep 相同
- 整理阶段：将所有存活对象向一端移动，然后清理边界外的内存

**优点**：没有内存碎片，内存利用率高
**缺点**：移动对象开销大，速度稍慢

#### ③ 引用计数（Reference Counting）— 已淘汰

- 跟踪每个对象被引用的次数，引用为 0 时回收
- **致命缺陷**：无法处理**循环引用**

```js
function fn() {
  let a = {};
  let b = {};
  a.ref = b;  // a 引用 b
  b.ref = a;  // b 引用 a
}
// 函数结束后 a、b 互相引用，引用计数不为 0，无法回收 ❌
```

现代浏览器已不再使用引用计数作为主算法。

---

## 2. 什么情况下会造成内存泄漏？常见场景

内存泄漏 = 不再需要的内存没有被 GC 回收。

### 六大常见场景

#### ① 意外的全局变量

```js
function leak() {
  // 没有声明，变成 window.leakData
  leakData = new Array(10000);
}
```

**解决**：始终使用 `let`/`const`/`var` 声明，开启严格模式 `"use strict"`

#### ② 被遗忘的定时器/回调

```js
setInterval(() => {
  // 持续引用外部变量，永远不会被回收
  const data = heavyObject;
  console.log(data);
}, 1000);
```

**解决**：不需要时 `clearInterval` / `clearTimeout`；组件卸载时清理

#### ③ 闭包持有不再需要的引用

```js
function createHandler() {
  const hugeData = new Array(100000).fill('*');
  const cache = fetchExpensiveData();
  return function () {
    // 确实使用了 hugeData，但 handler 用完后不再需要
    console.log(hugeData.length, cache.status);
  };
}
let handler = createHandler();
handler(); // 调用一次后就不再需要了
// 但 handler 变量仍持有闭包 → hugeData、cache 都无法回收
handler = null; // 手动解除引用
```

> **注意**：现代引擎（V8 等）会分析闭包内**实际使用的变量**，未使用的不会保留。真正的问题是闭包本身被长期持有（如存入全局数组、注册为事件回调），导致它引用的所有变量都无法释放。

**解决**：闭包不再需要时手动置为 `null`；避免将闭包存入长期存活的数据结构

#### ④ 未清理的 DOM 引用

```js
const elements = {};
function setup() {
  const el = document.getElementById('btn');
  elements.btn = el; // 即使 DOM 被移除，JS 仍引用它
}
```

**解决**：DOM 移除后，同步清除 JS 中的引用

#### ⑤ 未移除的事件监听器

```js
const btn = document.getElementById('btn');
btn.addEventListener('click', onClick);
// 元素被移除但监听器没有被移除，回调函数及其闭包无法回收
```

**解决**：`removeEventListener` 或使用 `AbortController`；组件卸载时清理

#### ⑥ 遗忘的 Map/Set 持有引用

```js
const map = new Map();
map.set(largeObject, 'data');
// 即使 largeObject 在其他地方不再需要，Map 仍持有它
```

**解决**：使用 `WeakMap` / `WeakSet`，键是弱引用，不阻止 GC

---

## 3. V8 引擎对垃圾回收做了哪些优化？

V8 是 Chrome 和 Node.js 的 JS 引擎，GC 优化非常多。

### ① 分代回收（Generational GC）

基于「大多数对象生命周期极短」的经验法则，将堆分为两代：

| | 新生代（Young Generation） | 老生代（Old Generation） |
|---|---|---|
| **大小** | 1～8 MB | 数百 MB～数 GB |
| **对象** | 新创建的、存活时间短 | 存活时间长的、大的 |
| **算法** | Scavenge（Cheney 算法） | Mark-Sweep + Mark-Compact |
| **GC 频率** | 频繁 | 较少 |
| **GC 速度** | 极快 | 较慢 |

### ② 新生代 — Scavenge 算法（即 Minor GC）

新生代分为两个等大的 **Semispace**（From 空间 + To 空间）：

1. 新对象分配在 From 空间
2. GC 时，将存活对象**复制**到 To 空间（紧凑排列，无碎片）
3. 清空整个 From 空间
4. From 和 To **角色互换**

**晋升条件**（满足任一即晋升到老生代）：
- 对象经历过一次 Scavenge 还存活
- To 空间使用率超过 25%

### ③ 老生代 — Mark-Sweep + Mark-Compact（即 Major GC）

- 主要使用 **Mark-Sweep**（速度快，但有碎片）
- 碎片过多时触发 **Mark-Compact**（整理内存，速度慢）

### ④ 并发标记（Concurrent Marking）— 减少主线程阻塞

传统 GC 的标记阶段需要**全停顿（Stop-The-World）**。V8 的优化：

- **增量标记（Incremental Marking）**：将标记工作拆成小块，穿插在 JS 执行之间
- **并发标记**：标记工作在**后台线程**执行，主线程继续跑 JS
- **并发清理**：清除阶段也完全在后台线程完成

**三色标记法**（并发标记的实现基础）：

V8 使用三色标记来追踪对象状态：

| 颜色 | 含义 |
|---|---|
| **白色** | 未被访问（潜在垃圾） |
| **灰色** | 已被访问，但其引用的对象**尚未全部扫描** |
| **黑色** | 已被访问，且其引用的对象**全部扫描完毕** |

标记过程：初始所有对象为白色 → 从 Root 开始，直接引用的对象标灰 → 灰色对象扫描完自己的引用后变黑，其引用标灰 → 直到没有灰色对象 → 剩余白色对象即为垃圾。

```
传统 GC：  |████ 全停顿 ████|→ JS 执行

V8 优化：  |标记| JS |标记| JS |标记| → 后台并发清理
          （增量）        （并发）
```

> **并发标记的难点**：JS 在标记过程中修改了引用关系怎么办？V8 使用**写屏障（Write Barrier）**机制——每当 JS 修改对象引用，写屏障会通知 GC 重新标记受影响的对象，确保不会误删。

### ⑤ 并发整理（Concurrent Compaction，V8 新版本）

老生代的 Mark-Compact 也在逐步实现并发化，进一步减少主线程阻塞时间。

### ⑥ 空闲时回收（Idle-time GC）

利用浏览器空闲时间（`requestIdleCallback`）执行 GC 任务，减少对用户交互的影响。

---

## 一句话总结

> JavaScript 只对**堆内存**做垃圾回收，栈内存随调用栈自动释放。GC 基于**可达性**（从 GC Root 出发遍历引用链）判断对象是否存活，核心算法是**标记-清除**；常见内存泄漏源于全局变量、定时器、闭包长期持有、DOM 引用、事件监听、Map/Set；V8 通过**分代回收（Minor GC + Major GC）+ 三色标记 + 并发/增量处理**三大策略大幅降低 GC 停顿时间。
