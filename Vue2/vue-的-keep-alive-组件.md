---
type: review
---
# Vue 的 keep-alive 组件

## 一、为什么需要 keep-alive

组件在切换时默认会被**销毁和重建**：

```
<router-view />  ← 每次路由切换，旧组件销毁、新组件重新创建
```

这带来两个问题：
1. **状态丢失** — 表单输入、滚动位置、分页等状态全部归零
2. **性能浪费** — 频繁请求接口、重新计算、重新渲染

`<keep-alive>` 就是用来**缓存不活动的组件实例**，避免反复销毁和重建。

---

## 二、基本用法

### 2.1 包裹动态组件

```vue
<keep-alive>
  <component :is="currentComponent" />
</keep-alive>
```

### 2.2 包裹路由出口

```vue
<keep-alive>
  <router-view />
</keep-alive>
```

### 2.3 配合路由的 meta 配置（按需缓存）

```js
// router/index.js
const routes = [
  {
    path: '/list',
    component: List,
    meta: { keepAlive: true }   // 需要缓存
  },
  {
    path: '/detail',
    component: Detail,
    meta: { keepAlive: false }  // 不需要缓存
  }
]
```

```vue
<keep-alive>
  <router-view v-if="$route.meta.keepAlive" />
</keep-alive>
<router-view v-if="!$route.meta.keepAlive" />
```

---

## 三、核心概念

### 3.1 缓存机制

`<keep-alive>` 内部维护了两个对象：

```js
// 简化源码（Vue 2.6）
export default {
  name: 'keep-alive',
  abstract: true,   // 抽象组件，不会出现在父组件链中

  created() {
    this.cache = Object.create(null)  // 缓存池 { key: VNode }
    this.keys = []                     // 缓存 key 列表（用于 LRU）
  },

  destroyed() {
    // 销毁所有缓存的组件实例
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  render() {
    // 1. 获取默认插槽中的第一个子组件
    const vnode = getFirstComponentChild(this.$slots.default)

    // 2. 获取组件选项
    const componentOptions = vnode && vnode.componentOptions

    if (componentOptions) {
      // 3. 生成缓存 key
      const key = getCacheKey(componentOptions)

      // 4. 命中缓存：复用 VNode，不重新创建
      if (this.cache[key]) {
        vnode.componentInstance = this.cache[key].componentInstance
        // 更新 keys 顺序（LRU：最近使用的放末尾）
        remove(this.keys, key)
        this.keys.push(key)
      } else {
        // 5. 未命中缓存：存入缓存
        this.cache[key] = vnode
        this.keys.push(key)

        // 6. 超出 max 时，淘汰最久未使用的
        if (this.max && this.keys.length > parseInt(this.max)) {
          pruneCacheEntry(this.cache, this.keys[0], this.keys, this._vnode)
        }
      }

      // 7. 标记为 keepAlive
      vnode.data.keepAlive = true
    }

    return vnode || (this.$slots.default && this.$slots.default[0])
  }
}
```

**关键点：**
- `cache` 是一个**对象**，key 是组件的 cid + tag，value 是 VNode
- `keys` 是一个**数组**，用于维护缓存顺序（LRU 淘汰策略）
- 命中缓存时，直接复用 VNode 的 `componentInstance`，**不重新创建**

### 3.2 LRU 淘汰策略

**LRU = Least Recently Used（最近最少使用）**

```
缓存池（max=3）：

初始:        keys = ['A', 'B', 'C']
访问 A:      keys = ['B', 'C', 'A']     ← A 移到末尾
新组件 D:    keys = ['C', 'A', 'D']     ← 淘汰 B（最久未使用）
```

### 3.3 缓存 key 的生成

```js
function getCacheKey(componentOptions) {
  // key = cid::tag
  // 例如：1::router-view
  return componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
}
```

如果多个相同组件需要分别缓存，需要给 `<router-view>` 加 `key`：

```vue
<keep-alive>
  <router-view :key="$route.fullPath" />
</keep-alive>
```

---

## 四、两个钩子函数

`<keep-alive>` 为被缓存组件提供了**专属生命周期钩子**：

### 4.1 activated（激活时）

```vue
<script>
export default {
  activated() {
    // 组件被激活时调用（从缓存中取出）
    // 此时 DOM 已经渲染完成
    console.log('组件被激活')
    this.fetchData()  // 可以在这里刷新数据
  }
}
</script>
```

**调用时机：**
- 首次挂载时，`mounted` 之后调用
- 从缓存中恢复时调用

### 4.2 deactivated（停用时）

```vue
<script>
export default {
  deactivated() {
    // 组件被停用时调用（进入缓存）
    // 此时 DOM 还在，但组件已被缓存
    console.log('组件被停用')
    this.stopTimer()  // 清理定时器、事件监听等
  }
}
</script>
```

**调用时机：**
- 组件被替换时调用（进入缓存）
- 组件销毁前（`<keep-alive>` 本身被销毁时）

### 4.3 执行顺序

```
首次渲染:   beforeCreate → created → beforeMount → mounted → activated
切换离开:   deactivated
切换回来:   activated（不会再走 created/mounted）
销毁:       deactivated → destroyed（仅 keep-alive 自身销毁时）
```

---

## 五、include / exclude / max

### 5.1 include — 只缓存匹配的

```vue
<!-- 字符串 -->
<keep-alive include="List,Detail">
  <router-view />
</keep-alive>

<!-- 正则（需要 v-bind） -->
<keep-alive :include="/List|Detail/">
  <router-view />
</keep-alive>

<!-- 数组 -->
<keep-alive :include="['List', 'Detail']">
  <router-view />
</keep-alive>
```

> 匹配的是组件的 `name` 选项，不是路由名称。

### 5.2 exclude — 不缓存匹配的

```vue
<keep-alive exclude="Login">
  <router-view />
</keep-alive>
```

> `include` 和 `exclude` 同时存在时，`exclude` 优先级更高。

### 5.3 max — 最大缓存数量

```vue
<keep-alive :max="10">
  <router-view />
</keep-alive>
```

超出 `max` 时，按 LRU 策略淘汰最久未使用的组件。

---

## 六、常见使用场景

### 6.1 列表页 → 详情页

```
场景：列表页滚动到第 50 条，点进详情页，返回后希望还在第 50 条

方案：缓存列表页
```

```vue
<!-- App.vue -->
<keep-alive include="List">
  <router-view />
</keep-alive>
```

### 6.2 Tab 页切换

```
场景：多个 Tab 页频繁切换，每个 Tab 都有大量数据和状态

方案：缓存所有 Tab 组件
```

```vue
<keep-alive>
  <component :is="activeTab" />
</keep-alive>
```

### 6.3 表单填写中途离开

```
场景：用户填写了一半表单，切到其他页面再回来

方案：缓存表单页面，回来后数据还在
```

---

## 七、注意事项和坑

### 7.1 缓存的是组件实例，不是数据

```vue
<script>
export default {
  data() {
    return {
      list: []
    }
  },
  activated() {
    // 从缓存恢复时，list 还是上次的数据
    // 如果需要刷新，必须在这里手动调用
    this.refreshList()
  }
}
</script>
```

### 7.2 内存问题

```js
// 缓存大量组件实例会占用内存
// 建议：
// 1. 只缓存必要的组件（include）
// 2. 设置合理的 max 值
// 3. 在 deactivated 中清理大数据
export default {
  deactivated() {
    this.bigList = []  // 释放大数据
  }
}
```

### 7.3 不会触发 created / mounted

```vue
<script>
export default {
  created() {
    // ⚠️ 只在首次创建时调用，从缓存恢复时不会再调用
    this.fetchData()
  },
  activated() {
    // ✅ 每次进入都会调用，适合做数据刷新
    this.fetchData()
  }
}
</script>
```

### 7.4 include 匹配的是组件 name

```vue
<!-- 组件定义 -->
<script>
export default {
  name: 'MyList',  // ← include 匹配的是这个名字
  // ...
}
</script>

<!-- 使用 -->
<keep-alive include="MyList">
  <router-view />
</keep-alive>
```

---

## 八、相关问题

### Q1：keep-alive 的作用和缓存原理是什么？

**作用：**

`<keep-alive>` 是 Vue 的内置抽象组件，用于**缓存不活动的组件实例**，而不是销毁它们。这样在组件切换时可以保留状态、避免重复渲染和接口请求。

**缓存原理：**

```
1. 内部维护 cache（对象）和 keys（数组）

2. 渲染时获取插槽中的子组件 VNode

3. 根据组件的 cid + tag 生成唯一 key

4. 查找缓存：
   ├── 命中 → 复用缓存的 componentInstance，不重新创建
   │         更新 keys 顺序（LRU：移到末尾）
   └── 未命中 → 存入 cache，加入 keys
                 超出 max 时淘汰 keys[0]（最久未使用）

5. 返回 VNode（标记 keepAlive = true）
```

**关键设计：**
- **LRU 淘汰策略** — 最近使用的放末尾，超出 max 时淘汰头部
- **抽象组件** — `abstract: true`，不会出现在父组件链、不渲染 DOM
- **VNode 复用** — 命中缓存时直接复用整个 VNode 树，跳过创建过程

---

### Q2：activated 和 deactivated 钩子的调用时机？

**activated（激活）：**
- 首次挂载时，在 `mounted` 之后调用
- 从缓存中恢复时调用（**不会**触发 `created` / `mounted`）
- 适合：刷新数据、重新开始轮询、恢复滚动位置

**deactivated（停用）：**
- 组件被替换进入缓存时调用
- `<keep-alive>` 本身被销毁前调用
- 适合：清理定时器、取消事件监听、释放大数据

**执行顺序：**

```
首次:     created → mounted → activated
离开:     deactivated
回来:     activated（仅此一个）
销毁:     deactivated → destroyed
```

---

### Q3：如何实现部分页面缓存？

**方案一：路由 meta + v-if（最常用）**

```vue
<keep-alive>
  <router-view v-if="$route.meta.keepAlive" />
</keep-alive>
<router-view v-if="!$route.meta.keepAlive" />
```

```js
// router/index.js
{
  path: '/list',
  component: List,
  meta: { keepAlive: true }
}
```

**方案二：include / exclude（推荐）**

```vue
<keep-alive include="List,Detail">
  <router-view />
</keep-alive>

<keep-alive exclude="Login">
  <router-view />
</keep-alive>
```

**方案三：动态 include**

```vue
<template>
  <keep-alive :include="cachedViews">
    <router-view />
  </keep-alive>
</template>

<script>
export default {
  computed: {
    cachedViews() {
      return this.$store.state.cachedViews  // 动态管理缓存列表
    }
  }
}
</script>
```

**对比：**

| 方案 | 优点 | 缺点 |
|------|------|------|
| meta + v-if | 灵活，可在路由配置中管理 | 需要两个 `<router-view>` |
| include/exclude | 简洁，一个标签搞定 | 依赖组件 name |
| 动态 include | 可运行时增删缓存 | 需要额外状态管理 |
