---
type: review
---
# React Router

## 一、路由基础概念

### 1. 什么是客户端路由

- 传统 Web：每次页面切换都向服务器请求完整 HTML
- SPA（单页应用）：页面不刷新，由 JS 动态切换视图，URL 变化但不触发完整页面请求
- **客户端路由**：浏览器 URL 变化时，由前端 JS 拦截并决定渲染哪个组件，而非向服务器请求新页面

### 2. 两种路由模式

浏览器提供了两种方式改变 URL 而不触发页面刷新：

| 模式 | API | URL 示例 | 依赖 |
|---|---|---|---|
| History 模式 | `history.pushState` / `replaceState` | `example.com/about` | 需要服务器配合 |
| Hash 模式 | `window.location.hash` | `example.com/#/about` | 无需服务器配置 |

### 3. 核心原理：History API

```js
// pushState 不会触发页面刷新，只改变 URL 并加入历史栈
history.pushState({ page: 'about' }, '', '/about');

// popstate 事件：用户点击浏览器前进/后退按钮时触发
window.addEventListener('popstate', (event) => {
  console.log('URL changed to:', window.location.pathname);
});
```

- `pushState(state, title, url)`：向历史栈推入一条记录，URL 变化但不刷新页面
- `replaceState(state, title, url)`：替换当前历史记录，不新增栈条目
- `popstate`：用户前进/后退时触发（`pushState` 本身不触发）

### 4. 核心原理：Hash 路由

```js
// 改变 hash 不会触发页面刷新
window.location.hash = '/about';

// hashchange 事件：hash 变化时触发
window.addEventListener('hashchange', () => {
  console.log('Hash changed to:', window.location.hash);
});
```

- `#` 后面的内容不会发送给服务器，仅浏览器端使用
- 兼容性极好，IE8+ 支持

---

## 二、HashRouter vs BrowserRouter

### BrowserRouter

- 基于 `history.pushState` / `replaceState`
- URL 干净：`example.com/users/123`
- **需要服务器配置**：所有路径都应返回 `index.html`，否则刷新会 404
- Nginx 示例：`try_files $uri $uri/ /index.html;`

### HashRouter

- 基于 `window.location.hash` + `hashchange` 事件
- URL 带 `#`：`example.com/#/users/123`
- **无需服务器配置**，`#` 后的内容不发送到服务器
- 缺点：URL 不美观，SEO 不友好

### 对比

| 特性 | BrowserRouter | HashRouter |
|---|---|---|
| URL 格式 | `/about` | `/#/about` |
| 服务器配置 | 需要（返回 index.html） | 不需要 |
| SEO | 友好 | 不友好 |
| 刷新行为 | 需服务器兜底 | 正常 |
| 兼容性 | IE10+（需 polyfill） | IE8+ |
| 推荐场景 | 生产环境，有后端配合 | 静态托管、无服务端控制 |

---

## 三、React Router v6 核心组件

### 1. `<Routes>` — 路由容器

- v6 中替代了 v5 的 `<Switch>`
- **排他性匹配**：只渲染第一个匹配的路由
- 必须是 `<Route>` 的父组件

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
</Routes>
```

### 2. `<Route>` — 路由规则

- 定义 `path` 和对应渲染的 `element`
- 支持嵌套路由（Route 嵌套 Route）
- v6 使用相对路径匹配，v5 需要完整路径

```jsx
// 嵌套路由
<Route path="/dashboard" element={<Dashboard />}>
  <Route index element={<Overview />} />        {/* /dashboard */}
  <Route path="settings" element={<Settings />} /> {/* /dashboard/settings */}
</Route>
```

### 3. `<Outlet>` — 嵌套路由出口

- 嵌套路由中，子路由渲染到父组件的 `<Outlet />` 位置
- 类似 Vue 的 `<router-view>`
- 父组件通过 `<Outlet />` 声明"子路由内容插在这里"

```jsx
function Dashboard() {
  return (
    <div>
      <Sidebar />
      <Outlet /> {/* 子路由（Overview 或 Settings）渲染在此处 */}
    </div>
  );
}
```

### 4. 路由匹配流程

```
URL: /dashboard/settings

<Routes>                    // 找到匹配的 Route 树
  <Route path="/dashboard" element={<Dashboard />}>   // 匹配
    <Route path="settings" element={<Settings />} />  // 匹配
  </Route>
</Routes>

渲染结果：
<Dashboard>
  <Outlet /> → <Settings />
</Dashboard>
```

---

## 四、路由导航方式

### 1. 声明式：`<Link>` / `<NavLink>`

```jsx
<Link to="/about">About</Link>

// NavLink 自动添加激活样式
<NavLink
  to="/about"
  className={({ isActive }) => isActive ? 'active' : ''}
>
  About
</NavLink>
```

- 替代 `<a>` 标签，不触发页面刷新
- `NavLink` 额外提供 `isActive`、`isPending` 状态

### 2. 命令式：`useNavigate`

```jsx
const navigate = useNavigate();

navigate('/about');             // 跳转
navigate(-1);                   // 后退一步
navigate('/login', { replace: true }); // 替换当前记录
```

- 替代 v5 的 `useHistory`
- 支持数字参数（前进/后退）

---

## 五、路由守卫（权限控制）

### 1. React Router 没有内置守卫

不同于 Vue Router 有 `beforeEach`，React Router 通过**组合模式**实现权限控制。

### 2. 核心思路：包装组件 + 条件渲染

```jsx
function ProtectedRoute({ children }) {
  const isAuth = useAuth(); // 自定义认证 hook

  if (!isAuth) {
    return <Navigate to="/login" replace />;
  }

  return children;
}
```

### 3. 使用方式

```jsx
<Routes>
  <Route path="/login" element={<Login />} />
  <Route
    path="/dashboard"
    element={
      <ProtectedRoute>
        <Dashboard />
      </ProtectedRoute>
    }
  />
</Routes>
```

### 4. 带重定向回原页面

```jsx
function ProtectedRoute({ children }) {
  const isAuth = useAuth();
  const location = useLocation();

  if (!isAuth) {
    // 把用户想去的页面存在 state 里，登录后跳回
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children;
}

// 登录成功后
function Login() {
  const navigate = useNavigate();
  const location = useLocation();
  const from = location.state?.from?.pathname || '/';

  const handleLogin = () => {
    // 登录逻辑...
    navigate(from, { replace: true });
  };
  // ...
}
```

### 5. 高阶组件模式（可选）

```jsx
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const isAuth = useAuth();
    if (!isAuth) return <Navigate to="/login" replace />;
    return <Component {...props} />;
  };
}

// 使用
<Route path="/admin" element={withAuth(AdminPanel)} />
```

---

## 六、其他重要概念

### 1. `useParams` — 获取动态参数

```jsx
<Route path="/users/:id" element={<User />} />

function User() {
  const { id } = useParams(); // { id: '123' }
}
```

### 2. `useSearchParams` — 查询参数

```jsx
function Search() {
  const [searchParams, setSearchParams] = useSearchParams();
  const query = searchParams.get('q'); // ?q=react

  setSearchParams({ q: 'vue' }); // 更新 URL
}
```

### 3. `useLocation` — 当前 URL 信息

```jsx
const location = useLocation();
// { pathname: '/about', search: '?id=1', hash: '', state: {...} }
```

### 4. `Navigate` 组件 — 声明式重定向

```jsx
<Route path="/old" element={<Navigate to="/new" replace />} />
```

### 5. `useRoutes` — JS 对象配置路由

```jsx
const routes = useRoutes([
  { path: '/', element: <Home /> },
  {
    path: '/dashboard',
    element: <Dashboard />,
    children: [
      { index: true, element: <Overview /> },
      { path: 'settings', element: <Settings /> },
    ],
  },
]);
```

---

## 七、回答问题

### Q1：HashRouter 和 BrowserRouter 的原理分别是什么？有什么区别？

**BrowserRouter 原理：**
- 基于 HTML5 History API（`pushState` / `replaceState`）
- 通过 `pushState` 改变 URL 而不触发页面刷新
- 监听 `popstate` 事件响应浏览器前进/后退
- URL 格式干净：`example.com/about`
- **必须服务器配合**：所有路径返回 `index.html`，否则刷新 404

**HashRouter 原理：**
- 基于 `window.location.hash` 和 `hashchange` 事件
- `#` 后的内容不发送到服务器，仅浏览器端使用
- URL 带 `#`：`example.com/#/about`
- 无需服务器配置，兼容性好

**核心区别：**

| 维度 | BrowserRouter | HashRouter |
|---|---|---|
| 底层 API | `history.pushState` | `location.hash` |
| URL 形式 | `/about` | `/#/about` |
| 服务器配置 | 必须配置兜底 | 不需要 |
| SEO | 友好 | 差 |
| 兼容性 | IE10+ | IE8+ |

---

### Q2：React Router v6 中 `<Routes>`、`<Route>`、`<Outlet>` 分别是什么作用？

- **`<Routes>`**：路由容器，替代 v5 的 `<Switch>`。提供**排他性匹配**，在所有子 `<Route>` 中只渲染第一个匹配的。必须包裹 `<Route>`。

- **`<Route>`**：定义路由规则。`path` 指定 URL 模式，`element` 指定渲染的组件。支持嵌套，子 Route 使用相对路径。`index` 属性标记默认子路由。

- **`<Outlet>`**：嵌套路由的**渲染出口**。父组件中放置 `<Outlet />`，子路由匹配的内容渲染到该位置。类似 Vue 的 `<router-view>`。

```jsx
// 完整示例
<Routes>
  <Route path="/dashboard" element={<Dashboard />}>  {/* 父路由 */}
    <Route index element={<Overview />} />            {/* 默认子路由 */}
    <Route path="settings" element={<Settings />} />  {/* 子路由 */}
  </Route>
</Routes>

// Dashboard 组件内
function Dashboard() {
  return (
    <div>
      <Sidebar />
      <Outlet />  {/* Overview 或 Settings 渲染在此 */}
    </div>
  );
}
```

---

### Q3：路由守卫（权限控制）怎么实现？

React Router **没有内置守卫机制**，通过**组合模式**实现：

**核心做法：包装组件 + 条件渲染 + `Navigate` 重定向**

```jsx
// 1. 创建守卫组件
function ProtectedRoute({ children }) {
  const isAuth = useAuth();
  const location = useLocation();

  if (!isAuth) {
    // 未登录 → 跳转登录页，保存目标路径
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children; // 已登录 → 正常渲染
}

// 2. 路由配置中包裹需要保护的路由
<Routes>
  <Route path="/login" element={<Login />} />
  <Route
    path="/dashboard"
    element={
      <ProtectedRoute>
        <Dashboard />
      </ProtectedRoute>
    }
  />
</Routes>

// 3. 登录成功后跳回原页面
function Login() {
  const navigate = useNavigate();
  const location = useLocation();
  const from = location.state?.from?.pathname || '/';

  const handleLogin = () => {
    doLogin();
    navigate(from, { replace: true });
  };
}
```

**进阶：角色权限**

```jsx
function RoleRoute({ children, roles }) {
  const { user } = useAuth();

  if (!user) return <Navigate to="/login" replace />;
  if (!roles.includes(user.role)) return <Navigate to="/403" replace />;

  return children;
}

// 使用
<RoleRoute roles={['admin', 'editor']}>
  <AdminPanel />
</RoleRoute>
```
