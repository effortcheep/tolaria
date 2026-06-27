---
type: review
---
# Vue 3 组件通信

## 一、组件通信概览

Vue 3 中组件之间的数据传递方式，按关系远近可分为：

| 场景 | 推荐方案 |
|------|----------|
| 父 → 子 | Props |
| 子 → 父 | Emit 事件 |
| 父 ↔ 子（双向） | v-model |
| 跨层级（爷孙等） | Provide / Inject |
| 兄弟组件 | 共同父组件提升状态 / 事件总线 / 全局状态管理 |
| 任意组件 | Pinia / Vuex（全局状态管理） |

---

## 二、Props（父 → 子）

### 基本用法

父组件通过 props 向子组件传递数据，这是**单向数据流**——数据只能从父组件流向子组件。

```vue
<!-- 父组件 -->
<ChildComponent :title="pageTitle" :count="num" />

<!-- 子组件 -->
<script setup>
// 选项式声明（推荐，有类型检查和默认值）
const props = defineProps({
  title: {
    type: String,
    required: true
  },
  count: {
    type: Number,
    default: 0
  }
})

// 或 TypeScript 类型声明
const props = defineProps<{
  title: string
  count?: number
}>()
</script>
```

### 关键规则

1. **单向数据流**：子组件不能直接修改 props。如需修改，应：
   - 用 `const localCopy = ref(props.value)` 创建本地副本
   - 或用 `emit` 通知父组件修改
2. **Props 验证**：支持 `type`、`required`、`default`、`validator`
3. **单向下行绑定**：父组件更新 → 子组件自动更新；反过来不行

---

## 三、Emit（子 → 父）

子组件通过 `emit` 触发自定义事件，父组件监听事件并处理数据。

```vue
<!-- 子组件 -->
<script setup>
const emit = defineEmits(['update', 'delete'])

function handleClick() {
  emit('update', { id: 1, name: 'new name' })
}
</script>

<!-- 父组件 -->
<ChildComponent @update="handleUpdate" />
```

### 关键点

- `defineEmits` 声明组件可触发的事件（推荐，有类型检查）
- 事件名推荐 kebab-case（`my-event`）或 camelCase（`myEvent`）
- 可以传任意数量的参数

---

## 四、v-model（双向绑定）

### 原生元素上的 v-model

`v-model` 是语法糖，本质是 `:value` + `@input` 的组合：

```vue
<input v-model="text" />
<!-- 等价于 -->
<input :value="text" @input="text = $event.target.value" />
```

### 组件上的 v-model

在组件上使用 `v-model`，Vue 3 中等价于：

```vue
<CustomInput v-model="title" />
<!-- 等价于 -->
<CustomInput :modelValue="title" @update:modelValue="title = $event" />
```

子组件需要：

```vue
<!-- 子组件 CustomInput -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

### 多个 v-model

Vue 3 支持为 v-model 指定参数名，实现多个双向绑定：

```vue
<UserName
  v-model:first-name="first"
  v-model:last-name="last"
/>

<!-- 子组件 -->
<script setup>
defineProps({
  firstName: String,
  lastName: String
})
defineEmits(['update:firstName', 'update:lastName'])
</script>
```

### v-model 修饰符

Vue 3 中组件可以接收 v-model 修饰符：

```vue
<MyComp v-model.capitalize="myText" />

<!-- 子组件中通过 modelModifiers 获取 -->
<script setup>
const props = defineProps({
  modelValue: String,
  modelModifiers: { default: () => ({}) }
})
const emit = defineEmits(['update:modelValue'])

function emitValue(e) {
  let value = e.target.value
  if (props.modelModifiers.capitalize) {
    value = value.charAt(0).toUpperCase() + value.slice(1)
  }
  emit('update:modelValue', value)
}
</script>
```

---

## 五、Provide / Inject（跨层级通信）

### 解决的问题

当组件层级很深（爷 → 父 → 子 → 孙），逐层传递 props 非常繁琐（"props 透传"问题）。Provide / Inject 允许**祖先组件向所有后代组件**直接提供数据，无论层级多深。

### 基本用法

```vue
<!-- 祖先组件 -->
<script setup>
import { ref, provide } from 'vue'

const theme = ref('dark')
provide('theme', theme) // 提供响应式数据

// 也可以提供方法
provide('changeTheme', (val) => {
  theme.value = val
})
</script>

<!-- 后代组件（任意层级） -->
<script setup>
import { inject } from 'vue'

const theme = inject('theme', 'light') // 第二个参数是默认值
const changeTheme = inject('changeTheme')
</script>
```

### 关键规则

1. **响应式**：`provide` 传递 `ref`/`reactive` 时，后代组件能保持响应式
2. **单向数据流**：默认情况下后代不应直接修改 inject 的值。如需修改，应由 provide 方提供修改方法
3. **非响应式值**：传递普通值（非 ref/reactive）时，后代拿到的是静态值
4. **Symbol 作为 key**：推荐用 Symbol 避免命名冲突：

```ts
// keys.ts
export const themeKey = Symbol('theme')

// 祖先
provide(themeKey, theme)

// 后代
const theme = inject(themeKey)
```

---

## 六、其他通信方式

### 1. ref / $parent（直接访问组件实例）

```vue
<!-- 父组件通过 ref 访问子组件 -->
<Child ref="childRef" />

<script setup>
import { ref, onMounted } from 'vue'
const childRef = ref(null)

onMounted(() => {
  childRef.value.someMethod() // 调用子组件暴露的方法
})
</script>
```

子组件需要通过 `defineExpose` 暴露：

```vue
<!-- 子组件 -->
<script setup>
function someMethod() { /* ... */ }
defineExpose({ someMethod })
</script>
```

> ⚠️ 不推荐滥用 ref 访问，会使父子耦合。优先用 props/emit。

### 2. Pinia（全局状态管理）

当多个不相关组件需要共享状态时，推荐使用 Pinia：

```ts
// stores/counter.ts
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: {
    double: (state) => state.count * 2
  },
  actions: {
    increment() { this.count++ }
  }
})
```

```vue
<!-- 任意组件中使用 -->
<script setup>
import { useCounterStore } from '@/stores/counter'
const counter = useCounterStore()
</script>

<template>
  <p>{{ counter.count }}</p>
  <button @click="counter.increment()">+1</button>
</template>
```

### 3. 事件总线（mitt）

Vue 3 移除了内置的 `$on`/`$emit`，可以用 `mitt` 库实现：

```ts
import mitt from 'mitt'
export const bus = mitt()

// 组件 A（发送）
bus.emit('message', 'hello')

// 组件 B（接收）
bus.on('message', (msg) => console.log(msg))
```

> ⚠️ 事件总线适合简单场景，复杂应用建议用 Pinia。

---

## 七、回答问题

### Q1：父子组件之间怎么通信？

**父 → 子**：通过 **Props** 传递数据

```vue
<!-- 父组件 -->
<Child :title="pageTitle" />

<!-- 子组件 -->
<script setup>
defineProps({ title: String })
</script>
```

**子 → 父**：通过 **Emit** 触发事件

```vue
<!-- 子组件 -->
<script setup>
const emit = defineEmits(['save'])
emit('save', data)
</script>

<!-- 父组件 -->
<Child @save="handleSave" />
```

**双向绑定**：通过 **v-model**

```vue
<!-- 父组件 -->
<Child v-model="value" />

<!-- 子组件内部通过 modelValue prop + update:modelValue emit 实现 -->
```

---

### Q2：跨层级组件（爷孙、更远）怎么通信？

**Provide / Inject** 是 Vue 3 中跨层级通信的首选方案：

- **祖先组件** `provide(key, value)` 提供数据
- **后代组件** `inject(key)` 注入数据，无论层级多深

```vue
<!-- 祖先 provide -->
<script setup>
const theme = ref('dark')
provide('theme', theme)
</script>

<!-- 后代 inject（任意深度） -->
<script setup>
const theme = inject('theme', 'light')
</script>
```

注意事项：
- 传递 `ref` 时保持响应式
- 推荐用 Symbol 作 key 避免冲突
- 如需修改，由 provide 方提供修改方法

如果需要全局共享状态（不限于组件树关系），推荐使用 **Pinia**。

---

### Q3：兄弟组件之间怎么通信？

**方案一：状态提升（推荐）**

将共享状态提升到共同父组件，通过 props 下发，通过 emit 回传：

```vue
<!-- 父组件 -->
<script setup>
const count = ref(0)
</script>

<template>
  <CompA :count="count" @increment="count++" />
  <CompB :count="count" />
</template>
```

**方案二：Pinia（全局状态）**

```ts
// stores/shared.ts
export const useSharedStore = defineStore('shared', {
  state: () => ({ data: null })
})
```

两个兄弟组件各自 import 同一个 store，读写同一份状态。

**方案三：事件总线（mitt）**

```ts
import { bus } from './eventBus'

// 组件 A
bus.emit('update', payload)

// 组件 B
bus.on('update', handler)
```

| 方案 | 适用场景 | 优缺点 |
|------|----------|--------|
| 状态提升 | 共享状态范围小，兄弟组件在同一父组件下 | 简单直接，但层级深时繁琐 |
| Pinia | 多处共享，状态复杂 | 官方推荐，DevTools 支持 |
| mitt 事件总线 | 一次性通知，无持久状态 | 简单但难追踪，大型应用不推荐 |

---

### Q4：Vue 3 的 v-model 在组件上怎么用？

Vue 3 的 `v-model` 在组件上是**语法糖**，本质是 prop + emit 的组合：

```vue
<!-- 父组件 -->
<CustomInput v-model="text" />

<!-- 等价于 -->
<CustomInput
  :modelValue="text"
  @update:modelValue="text = $event"
/>
```

子组件需要声明 `modelValue` prop 和 `update:modelValue` 事件：

```vue
<script setup>
defineProps({ modelValue: String })
defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="modelValue"
    @input="$emit('update:modelValue', $event.target.value)"
  />
</template>
```

**Vue 3 新特性：**

1. **多个 v-model**：用参数区分
   ```vue
   <Form v-model:name="name" v-model:email="email" />
   ```

2. **v-model 修饰符**：组件可通过 `modelModifiers` prop 获取修饰符
   ```vue
   <MyComp v-model.capitalize="text" />
   ```
