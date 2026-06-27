---
type: review
---
# Vue Router 的导航守卫

---

## 一、核心概念

### 1. 什么是导航守卫？

导航守卫（Navigation Guards）是 Vue Router 提供的**路由跳转拦截机制**，允许你在路由发生变化的各个阶段插入自定义逻辑，常用于：

- **权限控制**：未登录 → 跳转登录页
- **数据预加载**：进入页面前先获取数据
- **页面离开确认**：表单未保存时弹出确认框
- **埋点 / 日志**：记录页面访问

### 2. 守卫的本质

守卫的本质是**回调函数**，在路由跳转的特定时机被调用。守卫函数通常接收三个参数：

- `to`：即将进入的目标路由对象（`Route`）
- `from`：即将离开的当前路由对象（`Route`）
- `next`：**必须调用**的函数，用于决定是否放行（Vue Router 3.x）

> Vue Router 4.x（Vue 3）中 `next` 参数变为可选，可以通过返回值控制跳转（`return false` 阻止、`return { path: '/' }` 重定向），但 `next()` 仍然可用。

### 3. 路由解析的完整流程

理解导航守卫的执行顺序，需要先理解完整的路由解析流程：

```
触发导航（点击 <router-link> 或调用 $router.push）
        ↓
 ① 失活的组件里调用 beforeRouteLeave（组件内守卫）
        ↓
 ② 调用全局的 beforeEach（全局前置守卫）
        ↓
 ③ 在重用的组件里调用 beforeRouteUpdate（组件内守卫）
        ↓
 ④ 在路由配置里调用 beforeEnter（路由独享守卫）
        ↓
 ⑤ 解析异步路由组件（按需加载）
        ↓
 ⑥ 在被激活的组件里调用 beforeRouteEnter（组件内守卫）
        ↓
 ⑦ 调用全局的 beforeResolve（全局解析守卫）
        ↓
 ⑧ 导航被确认
        ↓
 ⑨ 调用全局的 afterEach（全局后置钩子）
        ↓
 ⑩ 触发 DOM 更新
        ↓
 ⑪ 调用 beforeRouteEnter 中传给 next 的回调函数
```

---

## 二、导航守卫的分类

### 1. 全局守卫

注册在 `router` 实例上，**所有路由跳转**都会触发。

| 守卫 | 调用时机 | 用途 |
| --- | --- | --- |
| `router.beforeEach` | 路由跳转前（最先触发） | 权限验证、登录检查 |
| `router.beforeResolve` | 所有组件内守卫和异步组件解析完成之后 | 需要确保数据加载完成 |
| `router.afterEach` | 路由跳转完成之后 | 埋点、页面标题修改 |

```js
// 全局前置守卫
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isAuthenticated()) {
    next('/login')   // 重定向到登录页
  } else {
    next()           // 放行
  }
})

// 全局解析守卫
router.beforeResolve((to, from, next) => {
  // 所有守卫和异步组件都已解析完毕
  next()
})

// 全局后置钩子（没有 next 参数）
router.afterEach((to, from) => {
  document.title = to.meta.title || 'App'
})
```

### 2. 路由独享守卫

注册在**单个路由配置**上，只有进入该路由时才触发。

```js
const routes = [
  {
    path: '/dashboard',
    component: Dashboard,
    beforeEnter: (to, from, next) => {
      // 只有访问 /dashboard 时才触发
      if (!hasPermission()) {
        next('/403')
      } else {
        next()
      }
    }
  }
]
```

### 3. 组件内守卫

注册在**组件实例**上，只在该组件相关的路由变化时触发。

| 守卫 | 调用时机 | `this` 可用？ |
| --- | --- | --- |
| `beforeRouteEnter` | 组件被创建前（进入前） | ❌ 不可用 |
| `beforeRouteUpdate` | 当前路由改变，但组件被复用时 | ✅ 可用 |
| `beforeRouteLeave` | 导航离开该组件时 | ✅ 可用 |

```js
export default {
  // 进入守卫：此时组件实例还没被创建
  beforeRouteEnter(to, from, next) {
    // this 不可用！
    // 通过 next(vm => { }) 回调访问组件实例
    next(vm => {
      console.log(vm.someData) // 组件已创建，可访问
    })
  },

  // 路由更新守卫：/user/1 → /user/2，组件被复用时触发
  beforeRouteUpdate(to, from, next) {
    // this 可用
    this.fetchUser(to.params.id)
    next()
  },

  // 离开守卫：常用于表单未保存确认
  beforeRouteLeave(to, from, next) {
    if (this.hasUnsavedChanges) {
      const answer = window.confirm('有未保存的更改，确认离开？')
      if (!answer) {
        next(false) // 取消导航
        return
      }
    }
    next() // 放行
  }
}
```

---

## 三、`next()` 函数的调用方式（Vue Router 3.x）

> **重要**：`next()` 是 Vue Router 3.x 的写法。Vue Router 4.x 推荐使用返回值，但 `next()` 仍然可用（不推荐混用）。

| 调用方式 | 含义 | 示例 |
| --- | --- | --- |
| `next()` | **放行**，进入下一个守卫或确认导航 | `next()` |
| `next(false)` | **取消导航**，URL 恢复到 `from` 路由 | `next(false)` |
| `next('/login')` | **重定向**到指定路径，触发新的导航 | `next('/login')` |
| `next({ path: '/login', query: { r: to.path } })` | **重定向**（对象形式），支持携带参数 | `next({ path: '/login', query: { r: to.path } })` |
| `next(error)` | **终止导航**并传递错误，触发 `router.onError()` 回调 | `next(new Error('无权限'))` |

### Vue Router 4.x 的替代写法

```js
// Vue Router 4.x 推荐用返回值代替 next()
router.beforeEach((to, from) => {
  if (to.meta.requiresAuth && !isAuthenticated()) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  // 不调用 next()，返回 undefined 或 true 表示放行
})
```

---

## 四、`beforeRouteEnter` 中 `next(vm => {})` 的特殊性

`beforeRouteEnter` 在组件创建**之前**调用，所以 `this` 不可用。通过 `next` 的回调参数可以在组件创建后访问实例：

```js
beforeRouteEnter(to, from, next) {
  // 此时 this 是 undefined
  next(vm => {
    // 组件已创建，vm 是组件实例
    vm.loadData()
  })
}
```

`next` 的回调函数会在导航确认、组件实例创建完成之后执行（对应流程图的第 ⑪ 步）。

---

## 五、完整的守卫执行示例

```js
// ===== 全局守卫 =====
router.beforeEach((to, from, next) => {
  console.log('1. 全局 beforeEach')
  next()
})

router.beforeResolve((to, from, next) => {
  console.log('5. 全局 beforeResolve')
  next()
})

router.afterEach((to, from) => {
  console.log('7. 全局 afterEach（无 next 参数）')
})

// ===== 路由配置 =====
const routes = [
  {
    path: '/page',
    component: Page,
    beforeEnter(to, from, next) {
      console.log('3. 路由独享 beforeEnter')
      next()
    }
  }
]

// ===== 组件内守卫 =====
// Page 组件
export default {
  beforeRouteEnter(to, from, next) {
    console.log('4. 组件 beforeRouteEnter')
    next()
  },
  beforeRouteUpdate(to, from, next) {
    console.log('4. 组件 beforeRouteUpdate')
    next()
  },
  beforeRouteLeave(to, from, next) {
    console.log('0. 组件 beforeRouteLeave（离开当前页时最先触发）')
    next()
  }
}
```

**首次进入 `/page` 的输出：**
```
1. 全局 beforeEach
3. 路由独享 beforeEnter
4. 组件 beforeRouteEnter
5. 全局 beforeResolve
7. 全局 afterEach
```

---

## 六、回答问题

### ① Vue Router 中有哪些导航守卫？按执行顺序排列出来。

**三类守卫、共 7 个：**

| 顺序 | 守卫 | 类型 | 调用时机 |
| --- | --- | --- | --- |
| 1 | `beforeRouteLeave` | 组件内 | 离开当前页面时（最先触发） |
| 2 | `beforeEach` | 全局 | 每次路由跳转前 |
| 3 | `beforeRouteUpdate` | 组件内 | 路由变化但组件被复用时 |
| 4 | `beforeEnter` | 路由独享 | 进入特定路由时 |
| 5 | `beforeRouteEnter` | 组件内 | 进入目标页面（组件创建前） |
| 6 | `beforeResolve` | 全局 | 所有守卫和异步组件解析完成后 |
| 7 | `afterEach` | 全局 | 路由跳转完成（无 next） |

### ② 全局前置守卫 `beforeEach` 的 `next()` 函数有哪些调用方式？分别代表什么意思？

5 种调用方式：

| 调用方式 | 含义 |
| --- | --- |
| `next()` | 放行，进入下一个守卫或确认导航 |
| `next(false)` | 取消导航，URL 恢复到 `from` 路由 |
| `next('/login')` | 重定向到指定路径，触发新的完整导航流程 |
| `next({ path: '/login', query: { ... } })` | 重定向（对象形式），可携带查询参数、hash 等 |
| `next(error)` | 终止导航并传入 Error 对象，触发 `router.onError()` 回调 |

> **注意**：`next()` 必须且只能被调用一次（`next(error)` 除外）。不调用会导致导航挂起；调用多次只有第一次生效。Vue Router 4.x 推荐用返回值替代 `next()`。
