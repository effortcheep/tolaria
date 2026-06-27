---
type: review
---
# 虚拟 DOM 与 Diff 算法

## 一、什么是虚拟 DOM（Virtual DOM）

虚拟 DOM 是对真实 DOM 树的 **JavaScript 对象映射**。它用轻量的 JS 对象描述 DOM 结构，不直接操作真实 DOM。

```
// 真实 DOM 节点 → 虚拟 DOM 对象
<div class="box" id="app">Hello</div>
↓
{
  tag: 'div',
  props: { class: 'box', id: 'app' },
  children: ['Hello']
}
```

## 二、为什么需要虚拟 DOM

| 痛点 | 虚拟 DOM 的解决方案 |
|------|-------------------|
| 直接操作 DOM 代价高（重排重绘） | 批量计算差异，最小化真实 DOM 操作 |
| 手动追踪状态变化容易出错 | 框架自动 diff，声明式编程 |
| 跨平台困难 | 虚拟节点可渲染到不同平台（浏览器、Native、SSR） |

> **核心价值**：不是"比直接操作 DOM 快"，而是提供 **声明式 + 高效更新** 的开发体验。

## 三、Diff 算法的核心思想

传统树 diff 算法复杂度为 **O(n³)**，不可接受。React/Vue 通过三个策略降到 **O(n)**：

### 1. 同层比较（Same Level）
只比较同一层级的节点，不做跨层移动。

```
旧:        新:
 A          A
/ \        / \
B   C      B   D    ← 只比较同层，发现 C→D
```

### 2. 同类型比较（Same Type）
节点类型不同 → 直接销毁重建，不继续递归比较子树。

```
旧: <div>     新: <span>
      ↓ 类型不同 → 整棵子树替换
```

### 3. Key 标识（Unique Key）
通过 `key` 属性识别哪些节点是"同一个"，实现高效复用。

```jsx
// React
<li key={item.id}>{item.name}</li>
```

## 四、Vue 2 的 Diff — 双端比较详解

Vue 2 的 patch 核心在 `src/core/vdom/patch.js` 中的 `updateChildren` 函数。

### 整体流程

```
patch(oldVnode, newVnode)
  ├─ 类型不同？ → 销毁旧节点，创建新节点
  ├─ 都是静态节点？ → 克隆
  └─ 都是同一节点？ → patchVnode（进入 diff）
       ├─ 文本节点？ → 直接更新 text
       ├─ 都有子节点？ → updateChildren（双端 diff）
       ├─ 只有新有子节点？ → 批量添加
       └─ 只有旧有子节点？ → 批量删除
```

### 双端比较 — 四个指针

```js
// 核心伪代码
while (oldStart <= oldEnd && newStart <= newEnd) {
  if (oldStartVnode == null) {
    oldStartVnode = oldCh[++oldStart]  // 跳过已处理节点
  } else if (oldEndVnode == null) {
    oldEndVnode = oldCh[--oldEnd]
  } else if (sameVnode(oldStartVnode, newStartVnode)) {
    // ① 头头比较 → 向后移动
    patchVnode(oldStartVnode, newStartVnode)
    oldStart++; newStart++
  } else if (sameVnode(oldEndVnode, newEndVnode)) {
    // ② 尾尾比较 → 向前移动
    patchVnode(oldEndVnode, newEndVnode)
    oldEnd--; newEnd--
  } else if (sameVnode(oldStartVnode, newEndVnode)) {
    // ③ 头尾比较 → 旧头移到尾部
    patchVnode(oldStartVnode, newEndVnode)
    insertBefore(parentElm, oldStartVnode.elm, oldEndVnode.elm.nextSibling)
    oldStart++; newEnd--
  } else if (sameVnode(oldEndVnode, newStartVnode)) {
    // ④ 尾头比较 → 旧尾移到头部
    patchVnode(oldEndVnode, newStartVnode)
    insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
    oldEnd--; newStart++
  } else {
    // ⑤ 四次都不命中 → 用 key 查找
    const idxInOld = oldKeyToIdx[newStartVnode.key]
    if (idxInOld === undefined) {
      // 旧列表中没有 → 新建节点插入
      createElm(newStartVnode)
    } else {
      // 找到了 → 移动到头部
      vnodeToMove = oldCh[idxInOld]
      patchVnode(vnodeToMove, newStartVnode)
      insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
      oldCh[idxInOld] = undefined  // 标记已处理
    }
    newStart++
  }
}
// 循环结束后：剩余旧节点删除，剩余新节点批量插入
```

### 图解示例

```
旧: [A, B, C, D, E]    新: [D, A, E, C, B]

第1轮: oldHead=A vs newHead=D → 不匹配
       oldTail=E vs newTail=B → 不匹配
       oldHead=A vs newTail=E → 不匹配
       oldTail=E vs newHead=D → 不匹配
       → 用 key 找 D，移到头部 → [D, A, B, C, E]

第2轮: oldHead=A vs newHead=A → 头头匹配 ✓
       → [D, A, B, C, E]

第3轮: oldHead=B vs newHead=E → 不匹配
       oldTail=E vs newTail=C → 不匹配
       oldHead=B vs newTail=E → 不匹配
       oldTail=E vs newHead=E → 尾头匹配 ✓
       → E 移到头部 → [D, A, E, B, C]

...以此类推
```

## 五、Vue 3 的 Diff — 最长递增子序列详解

Vue 3 的 diff 算法在 `packages/runtime-core/src/renderer.ts` 中，核心函数是 `patchKeyedChildren`。

### 整体流程

```
patchKeyedChildren(c1, c2, ...)
  ├─ 1. 从头部同步（相同前缀跳过）
  ├─ 2. 从尾部同步（相同后缀跳过）
  ├─ 3. 新节点多 → 挂载多余节点
  ├─ 4. 旧节点多 → 卸载多余节点
  └─ 5. 中间乱序部分 → 最长递增子序列优化
```

### 详细步骤

```js
// 第1步：从头开始，跳过相同前缀
// 旧: [A, B, C, D]  新: [A, B, E, F]
//      ↑↑ 跳过 A、B

// 第2步：从尾开始，跳过相同后缀
// 旧: [A, B, C, D]  新: [E, F, C, D]
//              ↑↑ 跳过 C、D

// 第3步：如果旧列表遍历完，新列表还有剩余 → 挂载
// 旧: [A, B]  新: [A, B, C, D]
//                   ↑↑ 新增 C、D

// 第4步：如果新列表遍历完，旧列表还有剩余 → 卸载
// 旧: [A, B, C, D]  新: [A, B]
//               ↑↑ 删除 C、D
```

### 第5步：中间乱序部分 — LIS 核心

```
旧: [A, B, C, D, E, F, G]
新: [A, E, B, D, C, H, F, G]
     ↑ 前缀 A 跳过        后缀 F,G 跳过 ↑

中间待处理:
旧: [B, C, D, E]     indices: 1,2,3,4
新: [E, B, D, C, H]

Step 1: 构建新列表的 key → index 映射
  { E→0, B→1, D→2, C→3, H→4 }

Step 2: 遍历旧中间节点，找它们在新列表中的位置
  B → newIndex=1
  C → newIndex=3
  D → newIndex=2
  E → newIndex=0

  newIndex 数组: [1, 3, 2, 0]
  对应 source:   [2, 4, 3, 1]  // 旧节点在新列表中的索引

Step 3: 对 source 计算最长递增子序列（LIS）
  source = [2, 4, 3, 1]
  LIS = [2, 4] 或 [2, 3] → 对应节点 B,D 或 B,C 不需要移动

Step 4: 从后往前遍历新列表
  - H → 旧列表中没有 → 新建
  - C → 不在 LIS 中 → 移动
  - D → 在 LIS 中 → 不动
  - B → 在 LIS 中 → 不动
  - E → 不在 LIS 中 → 移动
```

### 为什么要用 LIS？

```
核心思想：找到新列表中"相对顺序已经正确"的最长子集，
         这些节点不用移动，只移动其余节点 → 最小化移动次数

旧: [a, b, c, d]
新: [d, a, b, c]

newIndex: [3, 0, 1, 2]
LIS: [0, 1, 2] → a,b,c 保持不动
只需要移动 d → 1次移动（而非逐个移动）
```

### Vue 3 编译时优化

| 优化 | 原理 | 效果 |
|------|------|------|
| **PatchFlag** | 编译时给节点打标记（TEXT=1, CLASS=2, STYLE=4, PROPS=8...） | diff 时精确更新变化部分，跳过静态属性 |
| **静态提升** | 静态节点提取到 render 外部 | 不在每次 render 时重新创建 VNode |
| **Block Tree** | 只收集带 PatchFlag 的动态节点到扁平数组 | diff 只遍历动态节点，跳过整棵静态子树 |
| **事件缓存** | 内联事件函数缓存 | 避免不必要的子组件更新 |

```js
// 编译前
<div>
  <span>静态文本</span>
  <span>{{ msg }}</span>
</div>

// 编译后（简化）
const _hoisted_1 = createVNode("span", null, "静态文本")  // 静态提升

return createVNode("div", null, [
  _hoisted_1,
  createVNode("span", null, ctx.msg, PatchFlag.TEXT)  // 只标记文本
])
```

## 六、React 的 Diff — 单向遍历

### React 的三个假设（策略）

React 基于**三个前提假设**，将复杂度降到了 **O(n)**：

**策略一：跨层级移动极少，只做同层对比**

React 不会跨层级对比节点。如果两棵树在同一层级没有对应节点，React 不会去子树里找，而是直接**删除旧节点，创建新节点**。

```
旧树：          新树：
  A               A
 ├── B           ├── C
 │   └── D       │   └── D
 └── E           └── B
                      └── E
```

React 不会发现 B 和 E 移到了 C 下面。它看到 A 的第一个子节点从 B 变成了 C，就会**销毁 B 及其子树 D，新建 C 及其子树 D**。然后 A 的第二个子节点从 E 变成了 B，又销毁 E，新建 B 及其子树 E。

**这就是为什么用 CSS 的 `display: none` 控制显示隐藏，比用条件渲染（移除/添加 DOM 节点）性能更好**——避免节点被销毁重建。

**策略二：不同类型的元素产生不同的树**

如果节点的 `type` 变了（比如从 `<div>` 变成 `<span>`），React 不会尝试对比子节点，直接**销毁旧节点及整个子树，创建新节点及新子树**。

```javascript
// 旧
<div>
  <p>Hello</p>
</div>

// 新：div 变成了 span，整个子树都销毁重建
<span>
  <p>Hello</p>
</span>
```

只有 `type` 相同的节点才会继续对比 `props` 和 `children`。

**策略三：开发者可以通过 key 标识哪些元素是稳定的**

同一层级的子节点列表，通过 `key` 来判断哪些节点是"同一个"。

### Diff 的实际执行过程

```
1. 对比根节点
   ├── type 不同 → 销毁旧树，创建新树
   └── type 相同 → 对比 props
        ├── 对比属性（className、style、onClick 等）
        └── 对比 children
             ├── 子节点是单个 → 递归对比
             └── 子节点是列表 → 用 key 匹配，找出增/删/移动
```

### React 15: Reconciler（Stack Reconciler）

React 15 的 diff 是**同步递归**的，一旦开始就不可中断：

```
reconcileChildren(parentFiber, nextChildren)
  遍历新 children：
    ├─ 有旧 fiber 且 key/type 匹配 → 复用（移动位置）
    ├─ 有旧 fiber 但 type 不同 → 删除旧 fiber，创建新 fiber
    └─ 没有旧 fiber → 创建新 fiber
  遍历结束后：删除剩余未匹配的旧 fiber
```

### React 16+: Fiber Reconciler

React 16 引入 Fiber 架构，diff 变成**可中断的异步**过程：

```
render 阶段（可中断）：
  workLoop()
    while (nextUnitOfWork && !shouldYield()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    }

commit 阶段（同步，不可中断）：
  一次性应用所有 DOM 变更
```

### React diff 的两个阶段

```
Diff 算法（reconcileSingleChild / reconcileChildrenArray）

规则：
1. 只比较同层节点（不跨层）
2. 类型不同 → 整棵子树销毁重建
3. 通过 key + type 判断是否复用

列表 diff：
  从左到右遍历新列表：
    - 旧 fiber 的 key 和 type 都匹配 → 复用
    - 不匹配 → 标记删除，继续查找
  遍历完成后：标记剩余旧 fiber 为删除
```

### React 的"单向查找"策略

```
旧: [A, B, C, D, E]  (fiber linked list)
新: [A, C, E, B, D]

从左到右遍历新列表：
  A → 旧A 匹配 ✓ 复用
  C → 旧B 不匹配，向后找旧C → 找到 ✓ 复用
       （旧B 标记为待处理）
  E → 旧D 不匹配，向后找旧E → 找到 ✓ 复用
       （旧D 标记为待处理）
  B → 从 lastPlacedIndex 之后找旧B → 找到 ✓ 复用
  D → 找旧D → 找到 ✓ 复用

结果：复用所有节点，但需要移动 B 和 D
```

### React 为什么不使用双端比较？

```
React 团队的设计取舍：
1. 单向遍历实现更简单，代码更少
2. 配合 Fiber 的可中断特性，单向遍历更容易暂停/恢复
3. 实际业务中，列表头部插入/删除的概率比中间乱序小
4. 用 key 的 map 查找弥补效率损失
```

## 七、三大框架 Diff 算法对比

| 特性 | Vue 2 | Vue 3 | React |
|------|-------|-------|-------|
| **比较策略** | 双端比较 | 双端比较 + LIS | 单向遍历（左→右） |
| **遍历方向** | 两端→中间 | 两端→中间 | 单向 |
| **复用判断** | sameVnode (key + tag) | sameVNode (key + type) | key + type |
| **移动优化** | 逐个查找移动 | LIS 最小化移动 | lastPlacedIndex |
| **中断能力** | 不可中断 | 不可中断 | 可中断（Fiber） |
| **静态优化** | 无 | PatchFlag/Block Tree | 无（React Compiler 实验中） |
| **时间复杂度** | O(n) | O(n) | O(n) |
| **最坏场景** | 尾→头插入大量节点 | 中间乱序 | 列表反转 |

## 八、Key 的作用 — 深度解析

### 1. Key 的本质：节点的"身份证"

```jsx
// 没有 key → 框架只能按索引猜测
<ul>
  <li>苹果</li>    // index 0
  <li>香蕉</li>    // index 1
  <li>橙子</li>    // index 2
</ul>

// 插入"葡萄"到第一位：
<ul>
  <li>葡萄</li>    // index 0 → 框架认为这是"苹果"，更新文本
  <li>苹果</li>    // index 1 → 框架认为这是"香蕉"，更新文本
  <li>香蕉</li>    // index 2 → 框架认为这是"橙子"，更新文本
  <li>橙子</li>    // index 3 → 新建
</ul>
// 结果：3次文本更新 + 1次创建（而不是 1次插入）

// 有 key → 框架精确识别身份
<ul>
  <li key="grape">葡萄</li>   // 框架知道这是新节点 → 创建
  <li key="apple">苹果</li>   // 复用
  <li key="banana">香蕉</li>  // 复用
  <li key="orange">橙子</li>  // 复用
</ul>
// 结果：1次创建，0次更新
```

### 2. Key 影响的状态复用

```jsx
// 一个带内部状态的组件
function TextInput({ label }) {
  const [value, setValue] = useState('')
  return <input value={value} onChange={e => setValue(e.target.value)} placeholder={label} />
}

// 用 index 做 key 的问题：
// 列表: [{name: '姓名'}, {name: '邮箱'}]
// 用户在"姓名"输入框输入了 "张三"

// 在头部插入"电话"字段后：
<textInput key={0} label="电话" />   // 复用了"姓名"的状态 → 输入框里是 "张三"
<textInput key={1} label="姓名" />   // 复用了"邮箱"的状态 → 输入框里是 ""
<textInput key={2} label="邮箱" />   // 新建

// 用 id 做 key：
<textInput key="phone"  label="电话" />  // 新建
<textInput key="name"   label="姓名" />  // 复用 → 输入框里还是 "张三" ✓
<textInput key="email"  label="邮箱" />  // 复用
```

### 3. Key 与副作用（Side Effects）

```jsx
// 表单、动画、focus 状态等都会受影响

// 用 index 做 key 时，列表重排会导致：
// ❌ 错误的组件被卸载/挂载（触发 useEffect 清理/执行）
// ❌ focus 位置错乱
// ❌ 动画播放错位
// ❌ 受控组件状态混乱
```

### 4. 为什么不能用 index 做 key？

| 场景 | 用 index 做 key 的后果 |
|------|----------------------|
| **列表头部插入** | 所有节点的 key 都变了 → 全部重新创建/更新 |
| **列表头部删除** | 第一个节点被"复用"到错误的数据 → 状态错乱 |
| **列表排序** | 节点与数据对应关系全部错位 |
| **列表过滤** | 被过滤掉的节点可能"复用"到不同数据 |
| **有状态组件** | 内部 state（输入框内容、checkbox 状态）跟着 key 走，会挂到错误的组件上 |
| **有副作用组件** | useEffect 的清理/执行时机错误 |

```js
// 什么时候可以用 index？
// ✅ 列表是静态的，不会增删改排序
// ✅ 列表项没有内部状态
// ✅ 列表项没有副作用（useEffect）
// 但即便如此，用唯一 id 仍然是更好的习惯
```

### 5. 最佳实践

```jsx
// ✅ 用稳定的唯一标识
<li key={item.id}>{item.name}</li>
<li key={user.uuid}>{user.name}</li>

// ✅ 用内容的哈希（当没有 id 时）
<li key={`${item.category}-${item.name}`}>{item.name}</li>

// ❌ 不要用 index
<li key={index}>{item.name}</li>

// ❌ 不要用随机数
<li key={Math.random()}>{item.name}</li>  // 每次渲染都是新 key → 全部重建

// ❌ 不要用对象引用
<li key={item}>{item.name}</li>  // 对象每次渲染都是新引用
```
