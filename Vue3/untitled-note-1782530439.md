---
type: review
---

# Vue 3 的 Diff 算法做了哪些优化？和 Vue 2 的 Diff 有什么区别？

## 一、Vue 2 的 Diff 策略 — 双端比较

### 核心思想

Vue 2 的 diff 核心在 `updateChildren` 函数中，采用**双端比较**策略：同时从新旧子节点列表的**两端向中间**逼近，用四个指针（oldStart、oldEnd、newStart、newEnd）进行四次比较。

### 四次比较规则

```
① 头头比较：oldStart vs newStart → 匹配则指针后移
② 尾尾比较：oldEnd vs newEnd   → 匹配则指针前移
③ 头尾比较：oldStart vs newEnd → 匹配则旧头移到尾部
④ 尾头比较：oldEnd vs newStart → 匹配则旧尾移到头部
⑤ 四次都不命中 → 用 key 在旧列表中查找，找到则移动，找不到则新建
```

### 执行流程

```
while (oldStart <= oldEnd && newStart <= newEnd) {
  if (① 头头匹配)       patchVnode, oldStart++, newStart++
  else if (② 尾尾匹配)  patchVnode, oldEnd--, newEnd--
  else if (③ 头尾匹配)  patchVnode, 移动旧头到尾部, oldStart++, newEnd--
  else if (④ 尾头匹配)  patchVnode, 移动旧尾到头部, oldEnd--, newStart++
  else {
    // ⑤ 用 key 查找
    idxInOld = keyToIndex[newStartVnode.key]
    if (找到了) patchVnode, 移动到头部
    else        创建新节点插入
    newStart++
  }
}
// 循环结束：剩余旧节点删除，剩余新节点批量插入
```

### 图示

```
旧: [A, B, C, D]  新: [D, A, B, C]

第1轮: 头头 A vs D ✗, 尾尾 D vs C ✗, 头尾 A vs C ✗, 尾头 D vs D ✓
       → D 移到头部 → [D, A, B, C]

第2轮: 头头 A vs A ✓ → 不动
第3轮: 头头 B vs B ✓ → 不动
第4轮: 头头 C vs C ✓ → 不动
```

## 二、Vue 2 Diff 的性能问题

### 问题一：暴力查找开销

四次比较都不命中时，Vue 2 会**线性遍历**旧列表找 key：

```js
for (let i = oldStart; i <= oldEnd; i++) {
  if (oldCh[i].key === newStartVnode.key) { /* 找到了 */ }
}
```

**最坏情况**：每次都需要查找，总比较次数接近 O(n²)。

### 问题二：无法识别"哪些节点不需要移动"

Vue 2 的双端比较是**逐个处理**的，每次只关注当前四个指针位置的节点，没有全局视角：

```
旧: [A, B, C, D, E]  新: [E, D, C, B, A]  （完全反转）

Vue 2 的过程：
  - 找 E → 移到头部
  - 找 D → 移到头部
  - 找 C → 移到头部
  - 找 B → 移到头部
  → 4 次 DOM 移动
```

它不知道 C、D、E 在新列表中已经保持了相对顺序，无法**批量跳过**这些节点。

### 问题三：编译时无优化

Vue 2 的模板编译不区分静态节点和动态节点，每次 `render` 时所有节点都会被创建为 VNode 并参与 diff，包括完全不会变化的静态节点。

## 三、Vue 3 的优化方案

### 优化一：最长递增子序列（LIS）— 最小化 DOM 移动

Vue 3 的 diff 核心改为**从两端同步 + 中间乱序部分用 LIS 优化**。

#### 整体流程

```
patchKeyedChildren(c1, c2):
  ① 从头部同步 — 跳过相同前缀
  ② 从尾部同步 — 跳过相同后缀
  ③ 新节点多 → 挂载多余节点
  ④ 旧节点多 → 卸载多余节点
  ⑤ 中间乱序部分 → LIS 优化
```

#### LIS 核心过程

```
旧: [A, B, C, D, E]  新: [A, E, B, D, C]

Step 1: 跳过前缀 A

中间部分:
  旧: [B, C, D, E]     位置: 1,2,3,4
  新: [E, B, D, C]

Step 2: 建立 key → index 映射
  { E→0, B→1, D→2, C→3 }

Step 3: 遍历旧中间节点，计算在新列表中的位置
  B → newIndex=1, C → newIndex=3, D → newIndex=2, E → newIndex=0
  newIndex 数组: [1, 3, 2, 0]

Step 4: 对 newIndex 计算最长递增子序列
  LIS = [1, 2] → 对应节点 B、D 不需要移动

Step 5: 从后往前遍历新列表
  C → 不在 LIS → 移动
  D → 在 LIS   → 不动
  B → 在 LIS   → 不动
  E → 不在 LIS → 移动
```

**关键区别**：Vue 2 用四次比较逐个处理，Vue 3 通过 LIS **全局识别**哪些节点已经处于正确位置，只移动真正需要移动的节点。

### 优化二：PatchFlag — 编译时标记动态内容

Vue 3 的模板编译器在**编译时**为每个 VNode 打上标记，精确标识哪些部分是动态的：

```js
// 编译后（简化）
createVNode("span", null, ctx.msg, PatchFlag.TEXT)   // 只有文本是动态的
createVNode("div", { class: ctx.cls }, null, PatchFlag.CLASS)  // 只有 class 是动态的
```

| PatchFlag 值 | 含义 |
|-------------|------|
| 1 | TEXT（文本动态） |
| 2 | CLASS（class 动态） |
| 4 | STYLE（style 动态） |
| 8 | PROPS（属性动态，排除 class/style） |
| 16 | FULL_PROPS（有 key 的属性，需完整 diff） |

diff 时，带有 PatchFlag 的节点**只比较标记的部分**，跳过所有静态属性。没有 PatchFlag 的节点在 diff 阶段**直接跳过**。

### 优化三：Block Tree — 只收集动态节点

传统虚拟 DOM 的 diff 需要递归遍历整棵树。Vue 3 引入 Block 的概念，将**只有带 PatchFlag 的动态节点**收集到一个扁平数组中：

```
传统 diff：               Block Tree diff：
    div                       div
   / | \                     / | \
  p  span  p               p  span  p       ← 静态，跳过
     |                          |
    text                       text         ← 动态（PatchFlag=1）
    msg                        msg

需要遍历所有节点            只遍历 Block 中收集的动态节点
```

一个 Block 通常是一个模板根节点或带 `v-if`/`v-for` 的结构化指令节点。Block 内部维护一个 `dynamicChildren` 数组，diff 时**只遍历这个数组**，而非整棵树。

### 优化四：静态提升（Hoist Static）

完全静态的节点在编译时提取到 `render` 函数外部，**每次渲染直接复用**，不重新创建 VNode：

```js
// 编译前
<div>
  <span>静态文本</span>
  <span>{{ msg }}</span>
</div>

// 编译后
const _hoisted_1 = createVNode("span", null, "静态文本")  // 只创建一次

return createVNode("div", null, [
  _hoisted_1,                                              // 直接复用
  createVNode("span", null, ctx.msg, PatchFlag.TEXT)       // 每次重新创建
])
```

### 优化五：事件缓存

内联事件函数会被缓存，避免每次渲染都生成新的函数引用导致子组件不必要的更新：

```js
// 编译前
<button @click="count++">+1</button>

// 编译后
const _cache = ctx.$cache || (ctx.$cache = [])
_cache[0] || (_cache[0] = () => ctx.count++)

createVNode("button", { onClick: _cache[0] }, "+1")
```

## 四、Vue 2 vs Vue 3 Diff 对比

| 对比维度 | Vue 2 | Vue 3 |
|---------|-------|-------|
| **比较策略** | 双端比较（四次匹配） | 双端同步 + LIS 优化 |
| **移动优化** | 逐个查找移动 | LIS 识别不动节点，最小化移动 |
| **查找方式** | 线性遍历或 key map | key → index 映射（O(1) 查找） |
| **静态节点处理** | 每次都参与 diff | PatchFlag 跳过 / 静态提升复用 |
| **动态节点范围** | 整棵树递归遍历 | Block Tree 只遍历动态节点 |
| **编译时优化** | 无 | PatchFlag + 静态提升 + Block + 事件缓存 |
| **VNode 创建** | 每次 render 都创建所有 VNode | 静态 VNode 只创建一次 |

## 五、回答问题

### Q1：Vue 2 的 Diff 策略是什么？它有什么性能问题？

**策略：双端比较**

Vue 2 使用双端比较算法，同时从新旧子节点列表的两端向中间逼近。每轮用四个指针进行四次比较（头头、尾尾、头尾、尾头），四次都不命中时通过 key 在旧列表中查找。这种方式对头部插入、尾部插入、头部移到尾部等常见场景效率很高。

**性能问题：**

1. **暴力查找开销**：四次比较都不命中时，需要线性遍历旧列表找 key，最坏情况下总比较次数接近 O(n²)
2. **缺乏全局视角**：每次只看四个指针位置的节点，无法识别"哪些节点已经处于正确位置"，导致不必要的 DOM 移动
3. **无编译时优化**：静态节点每次渲染都创建 VNode 并参与 diff，浪费计算资源

### Q2：Vue 3 做了哪些具体的优化来解决这个问题？

**优化 1：最长递增子序列（LIS）**

Vue 3 先从两端同步跳过相同前缀和后缀，然后对中间乱序部分建立 key → index 映射，计算最长递增子序列。LIS 中的节点不需要移动，只移动其余节点，**最小化 DOM 移动次数**。

**优化 2：PatchFlag + Block Tree**

编译时给节点打标记（PatchFlag），标识动态内容类型；Block 将动态节点收集到扁平数组。diff 时**只遍历动态节点的动态属性**，完全跳过静态内容。

**优化 3：静态提升**

静态节点提取到 render 函数外部，每次渲染直接复用 VNode 对象，不重新创建。

**优化 4：事件缓存**

内联事件函数缓存，避免每次渲染生成新的函数引用，防止子组件不必要的更新。
