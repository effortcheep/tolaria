---
type: review
---
# Vue 2 的 Mixin（混入）

**① Mixin 是做什么的？怎么使用？**\
**② Mixin 有什么问题？为什么 Vue 3 推荐用 Composition API 替代它？**

---

## 一、什么是 Mixin

**Mixin 本质**：一个普通的 JavaScript 对象，包含组件的任意选项（data、methods、computed、生命周期钩子等）。

**核心思想**：将多个组件中**重复的逻辑**抽取到一个独立对象中，然后"混入"到各个组件中，实现代码复用。

```js
// 定义一个 Mixin
const myMixin = {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() { this.count++ }
  },
  mounted() {
    console.log('mixin mounted')
  }
}
```

---

## 二、使用方式

### 1. 局部混入（组件内）

在组件的 `mixins` 选项中引入：

```js
import myMixin from './myMixin'

export default {
  mixins: [myMixin],
  mounted() {
    console.log(this.count) // 0 — 来自 mixin
    this.increment()        // 可以直接调用 mixin 的方法
  }
}
```

### 2. 全局混入

使用 `Vue.mixin()` 注入到**所有组件**：

```js
Vue.mixin({
  created() {
    const options = this.$options
    if (options.myOption) {
      console.log(options.myOption)
    }
  }
})
```

> ⚠️ **全局混入影响极大**，会混入到每个组件实例，包括第三方组件。实际项目中很少使用，通常只用于插件开发。

### 3. 多个 Mixin 混入

```js
const mixinA = { methods: { foo() { console.log('A foo') } } }
const mixinB = { methods: { bar() { console.log('B bar') } } }

export default {
  mixins: [mixinA, mixinB],  // 数组形式，可混入多个
  methods: {
    baz() {
      this.foo()  // 来自 mixinA
      this.bar()  // 来自 mixinB
    }
  }
}
```

---

## 三、合并策略

当 Mixin 和组件有**同名选项**时，Vue 按以下规则合并：

| 选项类型 | 合并策略 | 说明 |
|---------|---------|------|
| `data` | **递归合并**，组件优先 | 对象深层合并，组件的 data 属性覆盖 mixin 的同名属性 |
| `methods`、`computed`、`components`、`directives` | **对象合并**，组件优先 | 同名方法/属性被组件的覆盖 |
| 生命周期钩子 | **合并为数组**，mixin 先执行 | 都会被调用，mixin 的钩子在组件的**之前**执行 |
| `watch` | **合并为数组**，mixin 先执行 | 同名 watcher 都会被调用 |
| `provide`、`inject` | **对象合并**，组件优先 | 与 methods 策略相同 |
| `el`、`propsData` | **组件优先** | 只取组件的值 |

### 合并策略代码示例

```js
const mixin = {
  data() {
    return { msg: 'from mixin', count: 0 }
  },
  methods: {
    foo() { console.log('mixin foo') }
  },
  mounted() {
    console.log('mixin mounted')
  }
}

export default {
  mixins: [mixin],
  data() {
    return { msg: 'from component' }  // 覆盖 mixin 的 msg
  },
  methods: {
    foo() { console.log('component foo') }  // 覆盖 mixin 的 foo
  },
  mounted() {
    console.log('component mounted')
    // 输出顺序：
    // 'mixin mounted'   ← mixin 先执行
    // 'component mounted'  ← 组件后执行
  }
}
```

---

## 四、典型使用场景

### 1. 复用通用逻辑

```js
// 表单验证 mixin
const formValidation = {
  data() {
    return { errors: {} }
  },
  methods: {
    validate() { /* 通用验证逻辑 */ },
    clearErrors() { this.errors = {} }
  }
}
```

### 2. 复用生命周期行为

```js
// 窗口监听 mixin
const windowResize = {
  data() {
    return { windowWidth: window.innerWidth }
  },
  methods: {
    handleResize() {
      this.windowWidth = window.innerWidth
    }
  },
  mounted() {
    window.addEventListener('resize', this.handleResize)
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
  }
}
```

### 3. 注入通用计算属性

```js
const userPermission = {
  computed: {
    isAdmin() { return this.$store.state.user.role === 'admin' },
    isLoggedIn() { return !!this.$store.state.user.token }
  }
}
```

---

## 五、Mixin 的核心问题

### 1. 命名冲突

多个 mixin 或 mixin 与组件之间可能产生**命名冲突**，且 Vue 不会发出警告：

```js
const mixinA = { methods: { doSomething() { /* A 的逻辑 */ } } }
const mixinB = { methods: { doSomething() { /* B 的逻辑 */ } } }

export default {
  mixins: [mixinA, mixinB],  // doSomething 被覆盖，无警告
  methods: {
    doSomething() { /* 组件自己的逻辑 */ }  // 最终执行的是这个
  }
}
```

### 2. 隐式依赖

mixin 内部可能依赖组件的特定属性或方法，但从 mixin 本身**看不出来**：

```js
const myMixin = {
  mounted() {
    this.fetchData()  // 依赖组件提供 fetchData 方法
    this.initMap(this.$refs.map)  // 依赖组件的 ref
  }
}
```

### 3. 来源不清晰

当一个组件混入多个 mixin 时，很难判断某个方法或属性来自哪里：

```js
export default {
  mixins: [mixinA, mixinB, mixinC, mixinD],
  methods: {
    // this.someMethod() 到底来自哪个 mixin？
  }
}
```

### 4. 数据来源难以追踪

```js
export default {
  mixins: [userMixin, permissionMixin, dataMixin],
  // this.total、this.list、this.loading 分别来自哪个 mixin？
}
```

### 5. 逻辑碎片化

同一个功能的逻辑可能被拆分到多个选项中，无法形成内聚的逻辑单元：

```js
const mouseTracking = {
  data() { return { mouseX: 0, mouseY: 0 } },
  methods: { updateMouse(e) { this.mouseX = e.pageX; this.mouseY = e.pageY } },
  mounted() { window.addEventListener('mousemove', this.updateMouse) },
  beforeDestroy() { window.removeEventListener('mousemove', this.updateMouse) }
  // 一个功能分散在 4 个选项中
}
```

---

## 六、Vue 3 Composition API 替代

### 1. 用 composable 函数替代 mixin

```js
// useMousePosition.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMousePosition() {
  const x = ref(0)
  const y = ref(0)

  function update(e) {
    x.value = e.pageX
    y.value = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }  // 显式返回，来源清晰
}
```

### 2. 在组件中使用

```vue
<script setup>
import { useMousePosition } from './useMousePosition'
import { useWindowSize } from './useWindowSize'

const { x, y } = useMousePosition()    // 来源一目了然
const { width } = useWindowSize()      // 来源一目了然
</script>
```

### 3. 对比表

| 维度 | Mixin | Composition API (composable) |
|-----|------|----------------------------|
| 命名冲突 | 可能，无警告 | 通过解构重命名解决 |
| 数据来源 | 不清晰，多个 mixin 混合 | 明确，解构赋值一目了然 |
| 隐式依赖 | mixin 内部依赖外部属性 | 函数参数显式传入 |
| 类型支持 | 差，TypeScript 推断困难 | 好，完整类型推断 |
| 逻辑聚合 | 按选项类型分散 | 按功能聚合在一个函数中 |
| 复用方式 | 对象混入 | 函数调用 + 返回值 |
| 可测试性 | 需要通过组件测试 | 可单独测试函数 |
| 复杂逻辑 | 难以维护 | 可组合多个 composable |

---

## 七、回答问题

### Q1：Mixin 是做什么的？怎么使用？

**作用**：Mixin 是 Vue 2 中实现**组件逻辑复用**的机制，将重复的选项（data、methods、computed、生命周期钩子等）抽取到独立对象中，混入到多个组件中。

**两种使用方式**：
- **局部混入**：`mixins: [myMixin]`，只影响当前组件
- **全局混入**：`Vue.mixin({...})`，影响所有组件实例（慎用）

**合并规则**：
- `data`：递归合并，组件优先
- `methods`/`computed`：对象合并，组件同名覆盖
- 生命周期钩子：合并为数组，mixin 的钩子**先执行**

---

### Q2：Mixin 有什么问题？为什么 Vue 3 推荐用 Composition API 替代它？

**五个核心问题**：

| 问题 | 表现 |
|-----|------|
| 命名冲突 | 多个 mixin 同名属性互相覆盖，无警告 |
| 隐式依赖 | mixin 内部依赖组件的属性/方法，外部看不出来 |
| 来源不清晰 | `this.xxx` 无法快速判断来自哪个 mixin |
| 逻辑碎片化 | 同一功能的代码分散在 data/methods/lifecycle 各处 |
| 类型推断差 | TypeScript 难以推断 mixin 注入的属性类型 |

**Composition API 的解决方案**：
- **显式返回值**：`const { x, y } = useMouse()` — 来源清晰
- **参数传入**：`useFetch(url)` — 无隐式依赖
- **逻辑聚合**：一个函数包含完整的功能逻辑
- **完整类型推断**：TypeScript 友好
- **可独立测试**：不需要通过组件来测试 composable 函数
