---
type: review
---
# v-if 和 v-show 的区别

## 一、v-if 的实现原理

- **条件渲染指令**：根据表达式的值来决定是否创建/销毁 DOM 元素
- 条件为 `false` → 元素完全从 DOM 中移除
- 条件为 `true` → 重新创建元素并插入 DOM
- 涉及完整的组件生命周期：`beforeCreate` → `created` → `beforeMount` → `mounted` → ... → `beforeUnmount` → `unmounted`

```html
<!-- 条件为 false 时，DOM 中不存在该元素 -->
<div v-if="isVisible">内容</div>
```

## 二、v-show 的实现原理

- **显示/隐藏指令**：无论条件如何，元素始终存在于 DOM 中
- 通过 CSS 的 `display` 属性来控制显示/隐藏
- 条件为 `false` → `display: none`
- 条件为 `true` → 移除 `display` 样式或恢复原值
- **不触发**组件的创建/销毁生命周期

```html
<!-- 条件为 false 时，DOM 中仍存在该元素，只是隐藏 -->
<div v-show="isVisible">内容</div>
```

## 三、实现原理的核心区别

| 维度 | v-if | v-show |
|------|------|--------|
| DOM 操作 | 条件变化时创建/销毁节点 | 始终保留节点，只修改 CSS |
| 生命周期 | 切换时触发创建/销毁钩子 | 不触发创建/销毁钩子 |
| 编译方式 | 生成条件分支代码 | 编译为 CSS display 切换 |
| 初始渲染 | 条件 false 时完全不渲染 | 始终渲染，通过 CSS 隐藏 |
| template 元素 | ✅ 可以使用 | ❌ 不能使用 |

## 四、性能特点对比

### 首次渲染

- **v-if**：条件为 `false` 时，不创建 DOM，渲染开销小
- **v-show**：无论条件如何都会渲染，初始开销较大

### 切换性能

- **v-if**：每次切换都有创建/销毁 DOM 的开销
- **v-show`：只切换 `display` 属性，开销极小（约 0.25ms）

### 适用条件频率

- **v-if**：条件很少改变时性能更好
- **v-show`：频繁切换时性能更好

## 五、使用场景

### 适用 v-if 的场景

1. **条件很少改变**：运行时条件不太可能改变
2. **权限控制**：根据权限决定元素是否渲染
3. **初始渲染条件通常为 `false`**：避免不必要的 DOM 创建
4. **需要配合 `v-else-if` / `v-else` 使用**：多条件分支渲染

```html
<div v-if="role === 'admin'">管理员面板</div>
<div v-else-if="role === 'editor'">编辑面板</div>
<div v-else>普通用户面板</div>
```

### 适用 v-show 的场景

1. **需要频繁切换**：Tab 切换、手风琴、轮播图等
2. **切换频率高**：避免频繁创建/销毁 DOM 的开销
3. **需要保持组件状态**：切换后保留输入框内容、滚动位置等

```html
<div v-show="activeTab === 'tab1'">标签1内容</div>
<div v-show="activeTab === 'tab2'">标签2内容</div>
```

## 六、v-for 和 v-if 的优先级（Vue 2 vs Vue 3）

### 问题背景

当 `v-for` 和 `v-if` 同时作用在同一个元素上时，会产生优先级问题：

```html
<!-- 这种写法在 Vue 2 和 Vue 3 中行为不同 -->
<li v-for="item in list" v-if="item.isActive">
  {{ item.name }}
</li>
```

### Vue 2 的优先级：v-for > v-if

- **编译结果**：`v-for` 先执行，每轮循环内部再判断 `v-if`
- **意味着**：即使大部分 item 不满足条件，也会遍历整个列表
- **性能问题**：列表很长但只有少数满足条件时，浪费大量遍历

```html
<!-- Vue 2 编译等价于 -->
<template v-for="item in list">
  <li v-if="item.isActive">{{ item.name }}</li>
</template>
```

### Vue 3 的优先级：v-if > v-for

- **编译结果**：`v-if` 先执行，条件不满足直接跳过整个循环
- **意味着**：条件不满足时不会执行任何遍历
- **注意变化**：Vue 3 中两者写在同一节点会产生**编译警告**，建议显式处理

```html
<!-- Vue 3 会产生警告，推荐改为 -->
<template v-if="shouldShowList">
  <li v-for="item in list" :key="item.id">{{ item.name }}</li>
</template>
```

### 优先级对比表

| 维度 | Vue 2 | Vue 3 |
|------|-------|-------|
| 同一节点优先级 | `v-for` > `v-if` | `v-if` > `v-for` |
| 编译行为 | 先循环再判断 | 先判断再循环 |
| 是否产生警告 | ❌ 不警告 | ⚠️ 编译警告 |
| 性能影响 | 循环次数 = 总列表长度 | 条件 false 时 0 次循环 |

### 最佳实践（两个版本通用）

**方式一：computed 过滤（推荐）**

```html
<!-- 模板只负责渲染，过滤逻辑在 JS 中 -->
<li v-for="item in activeItems" :key="item.id">
  {{ item.name }}
</li>
```

```js
const activeItems = computed(() => list.filter(item => item.isActive))
```

**方式二：外层 template 包裹 v-for，内层 v-if**

```html
<!-- 明确控制执行顺序：先循环再判断 -->
<template v-for="item in list" :key="item.id">
  <li v-if="item.isActive">{{ item.name }}</li>
</template>
```

**方式三：外层 v-if，内层 v-for**

```html
<!-- 整个列表都不需要渲染时使用 -->
<template v-if="shouldShowList">
  <li v-for="item in list" :key="item.id">{{ item.name }}</li>
</template>
```

### 使用建议总结

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 每项独立判断是否显示 | computed 过滤 + v-for | 避免模板中混用，逻辑清晰 |
| 整个列表显隐切换 | 外层 v-if + 内层 v-for | 条件 false 时零开销 |
| Vue 2 已有代码 | 保持 v-for + v-if | 符合 Vue 2 优先级预期 |
| Vue 3 新项目 | 避免同一节点混用 | 编译警告 + 行为不明确 |

## 七、常见面试问题

### Q1：v-if 和 v-show 的区别

**实现原理：**

- **v-if**：条件渲染指令，条件为 `true` 时创建 DOM 元素，为 `false` 时销毁 DOM 元素
- **v-show`：显示/隐藏指令，元素始终存在于 DOM 中，通过 CSS `display` 属性控制显隐

**表现差异：**

| 对比维度 | v-if | v-show |
|---------|------|--------|
| DOM 存在性 | 条件 false 时不存在于 DOM | 始终存在于 DOM |
| 生命周期 | 触发创建/销毁钩子 | 不触发创建/销毁钩子 |
| template 支持 | ✅ 支持 | ❌ 不支持 |
| 初始渲染 | false 时不渲染 | 始终渲染 |
| 切换开销 | 较大（创建/销毁） | 较小（切换 CSS） |

### Q2：什么场景下用 v-if，什么场景下用 v-show？为什么？

**用 v-if 的场景：**

- 条件很少改变（如权限控制、角色判断）
- 原因：避免始终保持 DOM 节点的内存占用，初始条件为 false 时渲染更快

**用 v-show 的场景：**

- 需要频繁切换（如 Tab 切换、手风琴）
- 原因：v-show 只切换 CSS display 属性（约 0.25ms），而 v-if 需要创建/销毁整个 DOM 子树（约 1.5ms+），频繁切换时 v-show 性能更优

**简单总结：**

> **运行时条件很少改变 → v-if**
> **需要频繁切换 → v-show**

### Q3：v-for 和 v-if 能否同时使用？Vue 2 和 Vue 3 有什么区别？

**能否同时使用：**

技术上可以，但**不推荐**在同一节点上混用，会产生性能问题或编译警告。

**Vue 2 的情况：**

- **优先级**：`v-for` > `v-if`（v-for 先执行）
- **行为**：每轮循环内部再判断 v-if，即使大部分 item 不满足条件也会遍历整个列表
- **结果**：无警告，但可能浪费性能

**Vue 3 的情况：**

- **优先级**：`v-if` > `v-for`（v-if 先执行）
- **行为**：条件不满足时直接跳过整个循环
- **结果**：会产生编译警告，建议显式处理

**最佳实践：**

| 场景 | 推荐写法 |
|------|----------|
| 每项独立判断 | `computed` 过滤 + `v-for` |
| 整个列表显隐 | 外层 `v-if` + 内层 `v-for` |
| Vue 2 已有代码 | 保持原样（符合 Vue 2 预期） |
| Vue 3 新项目 | 避免同一节点混用 |
