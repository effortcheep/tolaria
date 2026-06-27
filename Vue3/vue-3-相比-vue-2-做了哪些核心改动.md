---
type: review
---
# Vue 3 相比 Vue 2 做了哪些核心改动？

三个小问：\
**① 响应式系统的变化：Vue 2 用的是什么方案？Vue 3 换成了什么？为什么？**\
**② Vue 3 的 Composition API 相比 Options API 解决了什么问题？**\
**③ 你能说出至少3个 Vue 3 其他重要的新特性或改进吗？**

---

## 一、响应式系统重构

### 1.1 Vue 2 的 Object.defineProperty 方案

Vue 2 使用 `Object.defineProperty` 劫持对象属性的 getter/setter：

```js
// Vue 2 内部大致原理
function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      // 收集依赖（订阅者）
      track(target, key)
      return val
    },
    set(newVal) {
      if (newVal === val) return
      val = newVal
      // 触发更新（通知订阅者）
      trigger(target, key)
    }
  })
}
```

**Vue 2 的局限：**

| 问题 | 说明 |
|------|------|
| **无法检测新增/删除属性** | 必须用 `Vue.set()` / `Vue.delete()` |
| **无法检测数组下标修改** | `arr[0] = 'new'` 不触发更新 |
| **无法检测数组长度修改** | `arr.length = 0` 不触发更新 |
| **需要递归遍历所有属性** | 初始化时开销大，深层对象性能差 |
| **每个属性都要创建 dep 实例** | 内存占用较高 |

Vue 2 对数组的特殊处理：重写了 7 个变异方法（`push`、`pop`、`shift`、`unshift`、`splice`、`sort`、`reverse`），在调用时手动触发通知。

### 1.2 Vue 3 的 Proxy 方案

Vue 3 使用 ES6 的 `Proxy` 代理整个对象：

```js
// Vue 3 内部大致原理
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    track(target, key)
    return Reflect.get(target, key, receiver)
  },
  set(target, key, value, receiver) {
    const result = Reflect.set(target, key, value, receiver)
    trigger(target, key)
    return result
  },
  deleteProperty(target, key) {
    const result = Reflect.deleteProperty(target, key)
    trigger(target, key)
    return result
  }
})
```

**Proxy 的优势：**

| 优势 | 说明 |
|------|------|
| **支持新增/删除属性** | `deleteProperty` 拦截器天然支持 |
| **支持数组下标和长度** | `set` 拦截器自动覆盖 |
| **惰性代理** | 只在访问嵌套对象时才递归代理，初始化更快 |
| **不侵入原始对象** | 不需要修改原对象的属性描述符 |
| **支持 Map、Set 等集合类型** | 通过 `reactive()` 可以代理集合 |

### 1.3 为什么 Vue 3 要换 Proxy

```
根本原因：Object.defineProperty 是对"属性"的劫持，
         Proxy 是对"整个对象"的代理。
```

- `defineProperty` 在初始化时就需要遍历所有属性并递归处理，**初始化开销大**
- `Proxy` 是惰性代理，**访问到哪一层才处理哪一层**，深层数据初始化更快
- `defineProperty` 无法感知属性的新增/删除，Vue 2 只能用 API 补丁
- `Proxy` 是语言层面的标准方案，拦截能力更全面

**唯一的代价：** Proxy 不兼容 IE11，这是 Vue 3 放弃 IE 支持的根本原因。

---

## 二、Composition API vs Options API

### 2.1 Options API 的代码组织方式

Vue 2 使用选项式 API，按选项类型组织代码：

```js
export default {
  data() {
    return { count: 0, name: '' }
  },
  computed: {
    double() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ },
    fetchName() { /* ... */ }
  },
  mounted() {
    this.fetchName()
  },
  watch: {
    count(newVal) { /* ... */ }
  }
}
```

### 2.2 Options API 的问题

**问题 1：逻辑分散**
一个功能的代码分散在 `data`、`methods`、`computed`、`watch`、生命周期钩子等多个选项中，功能越大越难维护。

```
Feature A 的逻辑 → 分散在 data、methods、watch 中
Feature B 的逻辑 → 也分散在 data、methods、watch 中
                    ↓
            阅读代码需要反复上下跳转
```

**问题 2：逻辑复用困难**
- Mixins 命名冲突、来源不清晰、隐式依赖
- 无法灵活组合多个逻辑复用单元

**问题 3：TypeScript 支持差**
- Options API 依赖 `this` 上下文，类型推导困难
- `this.method()` 的类型需要额外声明

### 2.3 Composition API 的解决方案

```vue
<script setup>
import { ref, computed, onMounted, watch } from 'vue'

// ---- Counter 逻辑集中在一起 ----
const count = ref(0)
const double = computed(() => count.value * 2)
function increment() { count.value++ }
watch(count, (newVal) => { /* ... */ })

// ---- User 逻辑集中在一起 ----
const name = ref('')
async function fetchName() { /* ... */ }
onMounted(() => fetchName())
</script>
```

### 2.4 对比总结

| 维度 | Options API | Composition API |
|------|------------|-----------------|
| **代码组织** | 按选项类型分类 | 按逻辑功能分类 |
| **逻辑复用** | Mixins（问题多） | Composables（清晰灵活） |
| **TypeScript** | 类型推导差 | 完整类型推导 |
| **代码压缩** | this 上的属性无法压缩 | 变量名可正常压缩 |
| **学习曲线** | 直观易上手 | 需要理解响应式原理 |
| **适用场景** | 简单组件 | 复杂逻辑、逻辑复用 |
| **两种 API 是否共存** | ✅ 可以在同一个项目中混用 |  |

> Options API 并未被废弃，简单场景下依然好用。Composition API 是对 Options API 的**补充和增强**，不是替代。

---

## 三、性能优化

### 3.1 编译时优化

**静态提升（Hoist Static）**
- Vue 2：每次重新渲染都会重新创建所有 VNode，包括静态节点
- Vue 3：静态节点提升到渲染函数外部，只创建一次，后续直接复用

```js
// Vue 3 编译后（伪代码）
const _hoisted_1 = createElementVNode('div', null, 'Hello') // 只创建一次

function render() {
  return (_hoisted_1) // 直接复用
}
```

**PatchFlag 标记**
- Vue 2：Diff 时需要逐个属性对比，无法区分哪些会变
- Vue 3：编译时为每个节点打标记，Diff 时只检查标记的属性

```js
// PatchFlag 常量
// 1 = TEXT（文本会变）
// 2 = CLASS（class 会变）
// 4 = STYLE（style 会变）
// 8 = PROPS（其他属性会变）
// 16 = FULL_PROPS（有动态 key 的属性）
// 32 = NEED_HYDRATION（事件监听器）
// 64 = STABLE_FRAGMENT（稳定片段）
// 128 = KEYED_FRAGMENT（带 key 的片段）
// 256 = UNKEYED_FRAGMENT（不带 key 的片段）
// 512 = NEED_PATCH（ref、指令等）
// 1024 = DYNAMIC_SLOTS（动态插槽）
```

**Block Tree（区块树）**
- Vue 2：递归对比整棵 VNode 树
- Vue 3：动态节点被收集到一个"扁平数组"中，Diff 时只遍历这个数组

```
Vue 2: 递归整棵树 → O(完整树)
Vue 3: 遍历扁平数组 → O(动态节点数)，跳过所有静态内容
```

**事件缓存**
- Vue 2：每次渲染都创建新的事件处理函数
- Vue 3：缓存事件处理函数，避免子组件不必要的更新

### 3.2 运行时优化

| 优化项 | Vue 2 | Vue 3 |
|--------|-------|-------|
| 响应式初始化 | 递归遍历所有属性 | 惰性代理，按需递归 |
| Diff 算法 | 双端比较 | 最长递增子序列（LIS） |
| 模板编译 | 运行时编译 | AOT 编译优化 |
| Tree-shaking | 不支持 | 完全支持 |

---

## 四、Tree-shaking 支持

Vue 3 的 API 设计为可摇树（tree-shaking）的：

```js
import { ref, computed, watch } from 'vue'
// 只有导入的 API 会被打包，其余全部移除
```

- 内置组件（`<Transition>`、`<KeepAlive>` 等）未使用则不打包
- 指令（`v-model`、`v-show` 等）未使用则不打包
- 最终包体积可以更小

---

## 五、其他核心改动

### 5.1 TypeScript 重写

- Vue 2：源码用 Flow（Facebook 的类型检查器）编写
- Vue 3：源码完全用 TypeScript 重写，类型定义与源码同步维护
- 好处：更好的 IDE 支持、自动补全、类型检查

### 5.2 新的内置组件与 API

| 新增 | 说明 |
|------|------|
| **Teleport** | 将组件渲染到 DOM 的任意位置（如模态框、Toast） |
| **Suspense** | 异步组件的加载状态管理（实验性） |
| **Fragment** | 组件支持多根节点（无需包裹单一父元素） |
| **`<script setup>`** | 编译时语法糖，减少样板代码 |
| **`defineComponent`** | 提供完整的类型推导 |
| **`createApp`** | 替代 `new Vue()`，隔离应用实例 |

### 5.3 应用实例创建方式变化

```js
// Vue 2
new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')

// Vue 3
const app = createApp(App)
app.use(router)
app.use(store)
app.mount('#app')
```

好处：每个应用实例独立，插件和全局配置不会互相污染。

### 5.4 v-model 变化

```vue
<!-- Vue 2：一个组件只能有一个 v-model，绑定 value + input -->
<child v-model="title" />

<!-- Vue 3：绑定 modelValue + update:modelValue，支持多个 v-model -->
<child v-model="title" />
<child v-model:title="title" v-model:content="content" />
```

### 5.5 v-if 与 v-for 优先级变化

- Vue 2：`v-for` 优先级高于 `v-if`（同一元素上）
- Vue 3：`v-if` 优先级高于 `v-for`（不推荐在同一元素使用）

### 5.6 生命周期钩子变化

- `beforeDestroy` → `beforeUnmount`
- `destroyed` → `unmounted`
- 新增 `onRenderTracked` / `onRenderTriggered`（调试用）
- `setup()` 等价于 `beforeCreate` + `created`

### 5.7 移除的 API

| 移除 | 替代方案 |
|------|---------|
| `Vue.set()` / `Vue.delete()` | Proxy 自动支持 |
| `$on` / `$emit` / `$off`（事件总线） | mitt 库 |
| `Vue.mixin()` | Composables |
| `$children` | `ref` + `defineExpose` |
| `filters` | 计算属性或方法 |
| `.sync` 修饰符 | `v-model:prop` |
| `$listeners` | 合并到 `$attrs` |

---

## 六、Vue 2 与 Vue 3 核心改动总览

| 维度 | Vue 2 | Vue 3 |
|------|-------|-------|
| **响应式** | Object.defineProperty | Proxy |
| **API 风格** | Options API | Options + Composition API |
| **源码语言** | Flow | TypeScript |
| **包体积** | 整体打包 | Tree-shaking |
| **性能** | 编译时无优化 | 静态提升 + PatchFlag + Block Tree |
| **Diff 算法** | 双端比较 | LIS 最长递增子序列 |
| **组件根节点** | 单根节点 | Fragment 多根节点 |
| **应用创建** | `new Vue()` | `createApp()` |
| **全局 API** | `Vue.xxx()` | `app.xxx()` 实例级别 |
| **v-model** | 单个，value+input | 多个，modelValue+update |
| **事件总线** | `$on` / `$off` | 移除，用 mitt |
| **IE 支持** | 支持 IE11 | 不支持 IE |
| **Teleport** | 无 | 内置 |
| **Suspense** | 无 | 内置（实验性） |

---

## 七、回答问题

### Q1：响应式系统的变化：Vue 2 用的是什么方案？Vue 3 换成了什么？为什么？

**Vue 2 的方案：`Object.defineProperty`**
- 对对象的每个属性单独定义 getter/setter
- 初始化时递归遍历所有属性，逐一劫持
- 数组通过重写 7 个变异方法实现响应式

**Vue 3 的方案：`Proxy`**
- 创建一个代理对象，拦截对目标对象的所有操作（get/set/delete 等）
- 惰性代理：只有访问到嵌套对象时才递归代理
- 天然支持数组下标、长度、新增/删除属性、Map/Set 等

**为什么换？**

1. **`defineProperty` 的先天缺陷**：无法检测属性的新增/删除，Vue 2 只能靠 `Vue.set()` / `Vue.delete()` 补丁；数组的下标和长度修改也无法检测
2. **初始化性能**：`defineProperty` 在初始化时递归处理所有属性，深层对象开销大；`Proxy` 是惰性代理，按需递归
3. **语言标准**：`Proxy` 是 ES6 标准方案，拦截能力全面，是更"正确"的做法
4. **代价**：放弃 IE11 支持

> 一句话总结：`Object.defineProperty` 是"属性级别"的劫持，`Proxy` 是"对象级别"的代理，后者能力更全面、性能更好。

### Q2：Vue 3 的 Composition API 相比 Options API 解决了什么问题？

**核心答案：解决了逻辑代码分散和复用困难两个根本问题。**

| 问题 | Options API | Composition API |
|------|------------|-----------------|
| **逻辑分散** | 同一功能的代码分散在 data/methods/computed/watch/钩子中 | 同一功能的代码集中在一起 |
| **逻辑复用** | Mixins：命名冲突、来源不明、隐式依赖 | Composables：命名可控、来源清晰、显式返回 |
| **TypeScript** | 依赖 this 上下文，类型推导困难 | 函数式风格，完整类型推导 |
| **代码压缩** | this.xxx 的属性名无法被压缩 | 普通变量可正常压缩 |

**具体举例：**

```
Options API 中一个"鼠标位置追踪"功能：
→ 需要分散在 data()、methods、mounted、beforeDestroy 四个选项中

Composition API 中同一个功能：
→ 抽取为 useMousePosition() 函数，所有逻辑集中，任何组件复用
```

> 注意：Composition API 不是 Options API 的替代品，而是**补充**。简单组件用 Options API 清晰直观，复杂逻辑用 Composition API 更好维护。两者可以混用。

### Q3：你能说出至少 3 个 Vue 3 其他重要的新特性或改进吗？

**① 编译时性能优化**
- **静态提升**：静态 VNode 只创建一次，后续渲染直接复用
- **PatchFlag**：编译时标记动态内容，Diff 只检查标记的属性，跳过静态内容
- **Block Tree**：动态节点被收集到扁平数组，Diff 从 O(树) 降到 O(动态节点数)
- 三者结合，让 Vue 3 的更新性能远超 Vue 2

**② Tree-shaking 支持**
- API 设计为可摇树的，未使用的功能不会被打包
- 内置组件（Transition、KeepAlive 等）按需引入
- 最终产物更小，生产环境更高效

**③ Fragment（多根节点）**
- Vue 2 要求组件必须有且只有一个根节点，经常需要多余的 `<div>` 包裹
- Vue 3 支持多根节点，模板更干净，减少无意义的 DOM 层级

**④ Teleport**
- 将组件的 DOM 渲染到指定目标位置（如 `document.body`）
- 解决了模态框、Toast、Tooltip 等需要脱离当前 DOM 层级的需求
- 以前需要手动操作 DOM 或用第三方库，现在内置支持

**⑤ TypeScript 原生支持**
- 源码用 TypeScript 重写，类型定义与源码同步
- IDE 自动补全、类型检查更准确
- 开发体验大幅提升

**⑥ createApp 替代 new Vue()**
- 每个应用实例独立，插件/全局配置互不污染
- 支持同页面多应用共存

**其他值得提及的改进：**
- `<script setup>` 编译时语法糖，减少样板代码
- v-model 支持多个（`v-model:title`、`v-model:content`）
- 生命周期更名更语义化（`beforeDestroy` → `beforeUnmount`）
- 移除了 `$on` / `$off` 等事件总线 API，推荐 mitt
