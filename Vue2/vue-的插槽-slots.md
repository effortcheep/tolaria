---
type: review
---

# Vue 的插槽（Slots）

## 一、核心概念

### 1. 什么是插槽

插槽是 Vue 组件的**内容分发机制**，让父组件可以向子组件的指定位置插入任意内容（模板、组件、HTML）。

插槽解决的核心问题：**组件的结构不固定，部分内容由使用者决定**。

```html
<!-- 子组件：Card.vue -->
<template>
  <div class="card">
    <div class="card-header">
      <slot name="header">默认标题</slot>
    </div>
    <div class="card-body">
      <slot>默认内容</slot>
    </div>
  </div>
</template>

<!-- 父组件使用 -->
<Card>
  <template #header>
    <h2>自定义标题</h2>
  </template>
  <p>自定义正文内容</p>
</Card>
```

### 2. 插槽的本质

插槽的内容在**父组件作用域**中编译，编译后生成 `VNode` 传递给子组件。子组件在自己的模板中通过 `<slot>` 标签将这些 VNode 渲染到对应位置。

```
父组件编译 → 生成 slot VNode → 传递给子组件 → 子组件 <slot> 处渲染
```

---

## 二、Vue 2 中的三种插槽

### 1. 默认插槽（Default Slot）

没有 `name` 属性的 `<slot>`，一个组件只能有一个默认插槽。

```html
<!-- 子组件 Alert.vue -->
<template>
  <div class="alert">
    <slot>这是一条默认提示</slot>
  </div>
</template>

<!-- 父组件 -->
<Alert>
  <p>自定义提示内容</p>
</Alert>
```

### 2. 具名插槽（Named Slot）

通过 `name` 属性标识，可以有多个，父组件用 `v-slot:name` 或 `#name` 指定插入位置。

```html
<!-- 子组件 Layout.vue -->
<template>
  <div class="layout">
    <header><slot name="header"></slot></header>
    <aside><slot name="sidebar"></slot></aside>
    <main><slot></slot></main>
    <footer><slot name="footer"></slot></footer>
  </div>
</template>

<!-- 父组件 -->
<Layout>
  <template #header>
    <h1>页面标题</h1>
  </template>

  <template #sidebar>
    <nav>侧边栏</nav>
  </template>

  <!-- 默认插槽，省略 name -->
  <p>主内容区</p>

  <template #footer>
    <p>页脚信息</p>
  </template>
</Layout>
```

### 3. 作用域插槽（Scoped Slot）

子组件通过 `<slot>` 的属性向父组件**传递数据**，父组件通过 `v-slot="scope"` 接收。

```html
<!-- 子组件 UserList.vue -->
<template>
  <ul>
    <li v-for="user in users" :key="user.id">
      <slot :user="user" :index="index">
        {{ user.name }} <!-- 默认渲染 -->
      </slot>
    </li>
  </ul>
</template>

<script>
export default {
  data() {
    return {
      users: [
        { id: 1, name: 'Alice', role: 'admin' },
        { id: 2, name: 'Bob', role: 'user' },
      ]
    }
  }
}
</script>

<!-- 父组件：自定义每项的渲染方式 -->
<UserList>
  <template #default="{ user, index }">
    <span>{{ index + 1 }}. {{ user.name }} - {{ user.role }}</span>
  </template>
</UserList>
```

---

## 三、作用域插槽的深入解析

### 1. 为什么叫"作用域"

普通插槽的内容在**父组件作用域**编译，访问的是父组件的数据：

```html
<!-- parentVar 是父组件的 data -->
<Card>
  <p>{{ parentVar }}</p>
</Card>
```

作用域插槽的内容**也在父组件作用域编译**，但可以接收子组件传递的数据作为参数：

```html
<!-- slotProps 来自子组件，但模板在父组件编译 -->
<Card>
  <template #default="slotProps">
    <p>{{ slotProps.item }}</p>
  </template>
</Card>
```

### 2. 底层编译原理

```js
// 父组件编译后，作用域插槽生成一个函数
// _t = this._t (renderSlot 的简写)
this._t('default', null, { user: this.user, index: this.index })

// 子组件的 render 函数中：
// 编译 slot 标签时，将父组件传入的函数执行
scopedSlots._t('default', null, { user: this.user })
```

关键区别：
| | 默认插槽 / 具名插槽 | 作用域插槽 |
|---|---|---|
| 编译结果 | VNode | 返回 VNode 的**函数** |
| 编译时机 | 父组件初始化时 | 子组件渲染时**调用**函数 |
| 数据来源 | 父组件作用域 | 父组件作用域 + 子组件传入的参数 |

### 3. 使用场景

**场景一：列表组件自定义渲染**

```html
<!-- 表格组件：每列的渲染方式由使用者决定 -->
<DataTable :data="users">
  <template #column-name="{ row }">
    <a :href="`/user/${row.id}`">{{ row.name }}</a>
  </template>
  <template #column-status="{ row }">
    <Badge :type="row.active ? 'success' : 'error'">
      {{ row.active ? '活跃' : '禁用' }}
    </Badge>
  </template>
</DataTable>
```

**场景二：高级组件封装（如无限滚动）**

```html
<VirtualList :items="bigList" :item-height="50">
  <template #default="{ item, style }">
    <div :style="style">{{ item.name }}</div>
  </template>
</VirtualList>
```

**场景三：Render Props 模式**

```html
<!-- 提供数据，渲染逻辑交给使用者 -->
<Fetch url="/api/users">
  <template #default="{ data, loading, error }">
    <div v-if="loading">加载中...</div>
    <div v-else-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="u in data" :key="u.id">{{ u.name }}</li>
    </ul>
  </template>
</Fetch>
```

---

## 四、`v-slot` 的语法演变

| 版本 | 语法 | 说明 |
|---|---|---|
| Vue 2.5 之前 | `slot="name"` + `slot-scope="scope"` | 旧语法，已废弃 |
| Vue 2.6+ | `v-slot:name="scope"` | 统一新语法 |
| Vue 2.6+ | `#name="scope"` | `v-slot` 的缩写 |

```html
<!-- 旧语法（已废弃） -->
<UserList>
  <template slot="default" slot-scope="{ user }">
    {{ user.name }}
  </template>
</UserList>

<!-- 新语法 -->
<UserList>
  <template #default="{ user }">
    {{ user.name }}
  </template>
</UserList>
```

**注意**：`v-slot` 只能用在 `<template>` 或组件标签上，不能用在普通 HTML 元素上。

---

## 五、动态插槽名

```html
<template #[dynamicSlotName]>
  动态插槽内容
</template>

<script>
export default {
  data() {
    return {
      dynamicSlotName: 'header'  // 可以动态切换
    }
  }
}
</script>
```

---

## 六、插槽的 `this` 指向问题

```html
<!-- ❌ 错误：插槽内容访问子组件的 this -->
<Child>
  <p>{{ this.childData }}</p>
</Child>

<!-- ✅ 正确：作用域插槽接收子组件数据 -->
<Child>
  <template #default="{ childData }">
    <p>{{ childData }}</p>
  </template>
</Child>
```

---

## 七、回答问题

### ① Vue 2 中有几种插槽？分别怎么用？

**三种插槽**：

| 插槽类型 | 定义方式 | 使用方式 | 数据来源 |
|---|---|---|---|
| **默认插槽** | `<slot>`（无 name） | 直接在子组件标签内写内容 | 父组件 |
| **具名插槽** | `<slot name="xxx">` | `<template #xxx>` 或 `v-slot:xxx` | 父组件 |
| **作用域插槽** | `<slot :prop="value">` | `v-slot:xxx="scope"` 或 `#xxx="{ prop }"` | 子组件传入 + 父组件 |

核心区别在于**数据流向**：默认/具名插槽的内容完全由父组件决定；作用域插槽让子组件把数据"回传"给父组件，父组件拿到数据后再决定怎么渲染。

### ② 作用域插槽（Scoped Slots）是什么？它的使用场景是什么？

**本质**：子组件通过 `<slot>` 属性将内部数据暴露给父组件，父组件的模板虽然在父组件作用域编译，但能接收子组件传来的参数来决定渲染内容。

**编译层面**：普通插槽编译为 VNode，作用域插槽编译为**函数**（在子组件渲染时调用，返回 VNode）。

**核心使用场景**：
1. **列表/表格组件** — 使用者自定义每行/每列的渲染方式（如 `<DataTable #column-name="{ row }">`）
2. **数据获取组件** — 提供 `data / loading / error`，渲染逻辑由使用者决定（Render Props 模式）
3. **虚拟列表** — 提供 `item` 和 `style`，使用者决定每项的 DOM 结构
4. **弹窗/对话框** — 暴露 `close` 方法，使用者自定义底部按钮

一句话总结：当**子组件提供数据，父组件决定渲染**时，使用作用域插槽。
