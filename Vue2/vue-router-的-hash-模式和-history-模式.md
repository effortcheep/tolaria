---
type: review
---
# Vue Router 的 hash 模式和 history 模式

## 一、核心概念

### 什么是前端路由？

前端路由的本质：**在不刷新页面的前提下，根据 URL 变化切换视图**。

传统多页应用（MPA）中，每次 URL 变化都会向服务器请求新页面。前端路由的目标是把这个过程"拦截"下来，由前端 JavaScript 根据 URL 匹配组件并渲染，实现单页应用（SPA）的体验。

### 前端路由要解决的三个问题

1. **URL 变化时，如何感知？** — 监听浏览器事件
2. **URL 变化时，如何不刷新页面？** — 拦截默认行为
3. **URL 变化时，如何匹配组件？** — 路由表映射

hash 模式和 history 模式的核心区别就在于**用不同的浏览器 API 来解决前两个问题**。

---

## 二、Hash 模式

### 2.1 什么是 hash？

URL 的 hash（也叫 fragment identifier）是 `#` 及其后面的部分：

```
https://example.com/#/about
                 ^^^^^^^^^
                 这就是 hash
```

关键特性：
- **hash 不会被发送到服务器** — 浏览器向服务器请求时会自动去掉 `#` 及其后面的内容
- **hash 变化不会触发页面刷新** — 这是浏览器规范定义的行为
- **hash 变化会被记录到浏览器历史** — 可以前进/后退

### 2.2 监听机制：`hashchange` 事件

```javascript
window.addEventListener('hashchange', () => {
  console.log('hash 变化了:', window.location.hash)
  // 根据新的 hash 匹配路由，渲染对应组件
})
```

`hashchange` 事件在 `window.location.hash` 变化时触发，这是 Hash 模式的核心监听手段。

### 2.3 Vue Router 的 Hash 模式实现（`HashHistory`）

```javascript
// 核心实现思路（简化）
class HashHistory {
  constructor(router) {
    this.router = router
    // 确保 URL 有 hash，没有则默认 '#/'
    ensureSlash()
  }

  // 跳转：直接修改 location.hash
  push(location) {
    window.location.hash = createHref(location)
  }

  // 监听：hash 变化时触发路由匹配
  setupListener() {
    window.addEventListener('hashchange', () => {
      this.transitionTo(getHash(), onComplete)
    })
  }

  // 获取当前路径：从 hash 中提取
  getCurrentLocation() {
    return getHash()  // 去掉 #，返回路径部分
  }
}
```

**URL 示例：**
```
https://example.com/#/
https://example.com/#/about
https://example.com/#/user/123
https://example.com/#/settings?tab=profile
```

### 2.4 Hash 模式的优缺点

| 维度 | 说明 |
|------|------|
| ✅ 兼容性 | IE8+ 全部支持 |
| ✅ 无需服务端配置 | hash 不会发到服务器，服务器永远收到的是根路径 |
| ✅ 部署简单 | 任何静态服务器都能用 |
| ❌ URL 不美观 | 有 `#`，看起来像"页面内锚点" |
| ❌ SEO 不友好 | 搜索引擎对 hash 内容的抓取支持有限 |
| ❌ 锚点功能冲突 | `#` 本来是用来定位页面锚点的，hash 路由占用了这个能力 |

---

## 三、History 模式

### 3.1 什么是 History API？

HTML5 引入了一组 History API，允许 JavaScript **直接操作浏览器的会话历史记录**，而不触发页面刷新：

```javascript
// 压入一条新历史记录，URL 变化但不刷新
history.pushState({ page: 'about' }, '', '/about')

// 替换当前历史记录
history.replaceState({ page: 'about' }, '', '/about')

// 前进/后退
history.go(-1)
history.back()
history.forward()
```

### 3.2 核心区别：`pushState` vs `location.hash`

| 行为 | `location.hash` | `history.pushState` |
|------|-----------------|---------------------|
| URL 变化 | ✅ | ✅ |
| 不刷新页面 | ✅ | ✅ |
| 产生历史记录 | ✅ | ✅ |
| URL 格式 | `/path#/route` | `/route`（干净的 URL） |
| 数据发送到服务器 | ❌（hash 不发送） | ✅（URL 路径部分会发送） |

最后一行是关键区别 — **`pushState` 改变的 URL 路径会真正发到服务器**。

### 3.3 监听机制：`popstate` 事件

```javascript
window.addEventListener('popstate', (event) => {
  console.log('URL 变化了:', window.location.pathname)
  // 根据新的 pathname 匹配路由，渲染对应组件
})
```

⚠️ **重要细节**：`popstate` 事件**只在浏览器前进/后退时触发**，`pushState` / `replaceState` 本身**不会触发** `popstate`。

所以 Vue Router 在 History 模式下，需要同时做两件事：
1. **拦截 `<a>` 标签的点击**（或监听 `popstate`），主动调用 `pushState`
2. **监听 `popstate`**，处理浏览器前进/后退

### 3.4 Vue Router 的 History 模式实现（`HTML5History`）

```javascript
// 核心实现思路（简化）
class HTML5History {
  constructor(router) {
    this.router = router
  }

  // 跳转：调用 pushState，然后触发路由匹配
  push(location) {
    const path = createPath(location)
    history.pushState({ key: genKey() }, '', path)
    this.transitionTo(getLocation())  // 主动触发匹配
  }

  // 监听：popstate 处理浏览器前进/后退
  setupListener() {
    window.addEventListener('popstate', () => {
      this.transitionTo(getLocation())
    })
  }

  // 获取当前路径：从 pathname 中提取
  getCurrentLocation() {
    return getLocation()  // window.location.pathname + search
  }
}
```

**URL 示例：**
```
https://example.com/
https://example.com/about
https://example.com/user/123
https://example.com/settings?tab=profile
```

---

## 四、两种模式的核心流程对比

```
┌─────────────────────────────────────────────────────────────┐
│                        Hash 模式                            │
│                                                             │
│  用户点击 <router-link>                                      │
│       ↓                                                     │
│  Vue Router 修改 window.location.hash                       │
│       ↓                                                     │
│  浏览器触发 hashchange 事件（不刷新页面）                      │
│       ↓                                                     │
│  Vue Router 监听到变化，匹配路由，渲染组件                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      History 模式                           │
│                                                             │
│  用户点击 <router-link>                                      │
│       ↓                                                     │
│  Vue Router 调用 history.pushState(null, '', '/about')      │
│       ↓                                                     │
│  URL 变化，但不触发任何事件，不刷新页面                        │
│       ↓                                                     │
│  Vue Router 主动调用 transitionTo()，匹配路由，渲染组件       │
│                                                             │
│  ─── 或者用户点击浏览器前进/后退 ───                          │
│       ↓                                                     │
│  浏览器触发 popstate 事件                                    │
│       ↓                                                     │
│  Vue Router 监听到变化，匹配路由，渲染组件                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、完整对比表

| 维度 | Hash 模式 | History 模式 |
|------|-----------|-------------|
| URL 格式 | `/#/path` | `/path` |
| 监听方式 | `hashchange` | `popstate` + 主动调用 `pushState` |
| 需要服务端配置 | ❌ | ✅（必须配置 fallback） |
| 浏览器兼容 | IE8+ | IE10+（需 polyfill） |
| SEO | 较差 | 较好 |
| URL 美观 | ❌ | ✅ |
| 锚点功能 | ❌ 冲突 | ✅ 正常使用 |
| 服务器会收到什么 | 根路径 `/` | 完整路径 `/about` |

---

## 六、回答问题

### ① hash 模式和 history 模式分别是怎么工作的？URL 长什么样？

**Hash 模式：**

工作原理：利用 URL 的 hash 部分（`#` 后面的内容）来模拟路由。当 hash 变化时，浏览器触发 `hashchange` 事件，Vue Router 监听该事件并匹配对应路由组件。hash 不会发送到服务器，不会触发页面刷新。

```
URL: https://example.com/#/about
     https://example.com/#/user/123
```

**History 模式：**

工作原理：利用 HTML5 的 `history.pushState()` / `history.replaceState()` API 来修改 URL，然后主动触发路由匹配。浏览器前进/后退时通过 `popstate` 事件监听。URL 是真实的路径，变化时不刷新页面。

```
URL: https://example.com/about
     https://example.com/user/123
```

### ② history 模式有什么问题？部署时需要怎么处理？

**核心问题：** History 模式的 URL 是真实路径，浏览器会把它发给服务器。如果服务器没有对应的资源，就会返回 404。

**具体场景：**
1. 用户在 `https://example.com/about` 刷新页面
2. 浏览器向服务器请求 `/about` 这个路径
3. 服务器上没有 `/about` 这个文件，返回 404

而 Hash 模式不会有这个问题，因为 `/#/about` 发到服务器的只是 `/`。

**部署时的解决方案：**

| 服务器 | 配置方式 |
|--------|----------|
| **Nginx** | `try_files $uri $uri/ /index.html;` — 找不到文件时回退到 `index.html` |
| **Apache** | `FallbackResource /index.html` 或 `.htaccess` 配置 `mod_rewrite` |
| **Node (Express)** | `app.get('*', (req, res) => res.sendFile('index.html'))` |
| **GitHub Pages** | 添加 404.html，里面用 JS 重定向到 `index.html` 并携带原始路径 |
| **CDN / 对象存储** | 配置 404 回退到 `index.html` |

**原理：** 所有配置的核心思想都一样 — **当服务器找不到对应文件时，把 `index.html` 返回给浏览器**。浏览器拿到 `index.html` 后，Vue Router 从 `window.location.pathname` 中读取路径，匹配路由，渲染组件。

```nginx
# Nginx 完整示例
server {
    listen 80;
    server_name example.com;
    root /var/www/dist;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**其他注意事项：**
- 后端 API 接口路由需要避开前端路由的路径，否则会被 fallback 拦截（通常用 `/api/` 前缀区分）
- 如果用 History 模式且页面有 SSR，服务端也需要做路由匹配渲染，否则首屏会有闪烁
