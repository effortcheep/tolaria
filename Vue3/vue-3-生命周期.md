---
type: review
relationships:
  - "[[review]]"
---

# Vue 3 生命周期

---

## 一、组件实例的生命周期概览

Vue 组件从创建到销毁经历四个阶段：

```
创建（Setup）→ 挂载（Mount）→ 更新（Update）→ 卸载（Unmount）
```

每个阶段都有对应的**生命周期钩子**，让开发者在特定时机执行代码。

---

## 二、选项式 API vs 组合式 API 钩子对照表

| 阶段 | 选项式 API | 组合式 API（setup 内） | 说明 |
|------|-----------|----------------------|------|
| 创建 | `beforeCreate` | `setup()` 本身（或 `<script setup>` 顶层） | 实例初始化，data/methods 尚不可用 |
| 创建 | `created` | `setup()` 本身（或 `<script setup>` 顶层） | 响应式数据已就绪，DOM 尚未挂载 |
| 挂载前 | `beforeMount` | `onBeforeMount()` | DOM 即将挂载，尚未渲染到页面 |
| **挂载后** | **`mounted`** | **`onMounted()`** | DOM 已挂载，可访问真实 DOM |
| 更新前 | `beforeUpdate` | `onBeforeUpdate()` | 响应式数据变化后，DOM 更新前 |
| **更新后** | **`updated`** | **`onUpdated()`** | DOM 已重新渲染完成 |
| 卸载前 | `beforeUnmount` | `onBeforeUnmount()` | 组件即将卸载，仍可访问 DOM |
| **卸载后** | **`unmounted`** | **`onUnmounted()`** | 组件已销毁，清理副作用 |
| 缓存组件 | `activated` | `onActivated()` | `<KeepAlive>` 缓存的组件被激活 |
| 缓存组件 | `deactivated` | `onDeactivated()` | `<KeepAlive>` 缓存的组件被停用 |
| 错误捕获 | `errorCaptured` | `onErrorCaptured()` | 捕获后代组件的错误 |
| 服务端渲染 | `serverPrefetch` | `onServerPrefetch()` | SSR 中异步获取数据 |

---

## 三、`setup()` 的特殊地位

### 3.1 setup 等价于 beforeCreate + created

```js
// 选项式 API
export default {
  beforeCreate() {
    console.log('beforeCreate')
  },
  created() {
    console.log('created')
  }
}

// 组合式 API — setup 执行时机覆盖了上面两个
export default {
  setup() {
    // 此处等价于 beforeCreate 和 created 之间的时机
    console.log('setup 执行，data/methods 已初始化')
  }
}
```

**关键结论：在组合式 API 中，不需要 `onBeforeCreate` 和 `onCreated`，因为 `setup()` 本身就是这个时机。**

### 3.2 `<script setup>` 顶层代码 = setup 函数

```vue
<script setup>
// 下面的代码在组件创建时执行，等价于 created
import { ref, onMounted } from 'vue'

const count = ref(0) // 创建时就执行

onMounted(() => {
  // 挂载后才执行
  console.log('mounted')
})
</script>
```

---

## 四、各钩子详解与使用场景

### 4.1 onMounted

**时机：** 组件 DOM 已渲染到页面后

**使用场景：**
- 操作 DOM（测量尺寸、第三方库初始化）
- 发起 API 请求（最常见的数据获取时机）
- 添加事件监听（需在 onUnmounted 中移除）

```js
import { ref, onMounted, onUnmounted } from 'vue'

const el = ref(null)

onMounted(() => {
  // DOM 已存在，可以安全访问
  console.log(el.value) // 真实 DOM 元素
  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  // 清理副作用
  window.removeEventListener('scroll', handleScroll)
})
```

### 4.2 onUpdated

**时机：** 响应式数据变化导致 DOM 重新渲染后

**注意：** 不要在回调中修改状态，可能导致无限循环

```js
onUpdated(() => {
  // DOM 已根据最新数据重新渲染
  console.log('DOM 已更新')
})
```

### 4.3 onBeforeUnmount / onUnmounted

**时机：** 组件即将被销毁 / 已销毁

**使用场景：** 清理定时器、取消订阅、移除事件监听

```js
const timer = setInterval(() => { /* ... */ }, 1000)

onBeforeUnmount(() => {
  clearInterval(timer) // 避免内存泄漏
})
```

---

## 五、完整生命周期执行流程

```
组件创建
  │
  ├── setup() 执行（等价于 beforeCreate + created）
  │     ├── 响应式数据初始化
  │     ├── 函数定义
  │     └── 注册生命周期钩子（onMounted 等，只是注册，不立即执行）
  │
  ├── onBeforeMount()  — DOM 即将挂载
  │
  ├── 渲染函数执行，生成真实 DOM
  │
  ├── onMounted()  — DOM 已挂载 ✓（钩子回调在此时才执行）
  │
  │  ── 数据变化 ──
  ├── onBeforeUpdate()  — DOM 即将更新
  │
  ├── DOM 重新渲染
  │
  ├── onUpdated()  — DOM 已更新
  │
  │  ── 组件卸载 ──
  ├── onBeforeUnmount()  — 即将卸载
  │
  ├── 组件销毁，事件监听移除
  │
  └── onUnmounted()  — 已卸载
```

---

## 六、父子组件生命周期执行顺序

### 6.1 挂载顺序

**规则：由外到内创建，由内到外挂载**

```
父 setup()
父 onBeforeMount()
  子 setup()
  子 onBeforeMount()
  子 onMounted()    ← 子组件先挂载完成
父 onMounted()      ← 父组件后挂载完成
```

**记忆口诀：** 父亲先进场（beforeMount），儿子先坐稳（mounted），父亲再坐稳。

### 6.2 卸载顺序

**规则：由外到内卸载**

```
父 onBeforeUnmount()
  子 onBeforeUnmount()
  子 onUnmounted()    ← 子组件先销毁
父 onUnmounted()      ← 父组件后销毁
```

### 6.3 更新顺序

```
父 onBeforeUpdate()
  子 onBeforeUpdate()
  子 onUpdated()    ← 子组件先更新完成
父 onUpdated()      ← 父组件后更新完成
```

---

## 七、回答问题

### Q1：Vue 3 的组合式 API 有哪些生命周期钩子？分别对应选项式 API 的哪个？

| 组合式 API | 选项式 API |
|-----------|-----------|
| `setup()` 顶层代码 | `beforeCreate` + `created` |
| `onBeforeMount()` | `beforeMount` |
| `onMounted()` | `mounted` |
| `onBeforeUpdate()` | `beforeUpdate` |
| `onUpdated()` | `updated` |
| `onBeforeUnmount()` | `beforeUnmount` |
| `onUnmounted()` | `unmounted` |
| `onActivated()` | `activated` |
| `onDeactivated()` | `deactivated` |
| `onErrorCaptured()` | `errorCaptured` |
| `onServerPrefetch()` | `serverPrefetch` |

**核心变化：** Vue 3 用 `on` 前缀命名，且去掉了 `beforeCreate` / `created`，因为 `setup()` 已覆盖这个时机。

---

### Q2：`onMounted` 和 `created` 的区别是什么？（setup 中的代码等价于哪个？）

**区别：**

| | `created` / `setup()` | `onMounted` |
|--|----------------------|-------------|
| 时机 | 组件实例创建完成 | DOM 已挂载到页面 |
| DOM 可访问 | ❌ 不可以 | ✅ 可以 |
| 数据请求 | ✅ 可以（更早拿到数据） | ✅ 可以（DOM 已就绪） |
| 操作 DOM | ❌ | ✅ |

**setup 中的顶层代码等价于 `created`。**

```vue
<script setup>
// ===== 这部分等价于 created =====
import { ref, onMounted } from 'vue'
const data = ref(null)
const res = await fetch('/api/data')  // 创建时就执行
data.value = res

// ===== 这部分等价于 mounted =====
onMounted(() => {
  console.log('DOM 已挂载，可以操作 DOM')
})
</script>
```

---

### Q3：父子组件的生命周期执行顺序是什么？

**挂载（从无到有）：**
```
父 setup → 父 onBeforeMount → 子 setup → 子 onBeforeMount → 子 onMounted → 父 onMounted
```

**卸载（从有到无）：**
```
父 onBeforeUnmount → 子 onBeforeUnmount → 子 onUnmounted → 父 onUnmounted
```

**原理：**
- 挂载时，父组件渲染到子组件时才会创建子组件，所以子组件的 `setup` 后执行；但子组件的 DOM 先完成渲染，所以子 `mounted` 先触发
- 卸载时，父组件先收到卸载通知（`beforeUnmount`），然后递归卸载子组件，最后父组件完成卸载
