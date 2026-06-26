---
type: review
related_to: "[[JavaScript 事件循环（Event Loop）]]"
---
# Promise 与 async/await

## 什么是 Promise

Promise 是 JavaScript 中处理**异步操作**的对象，代表一个最终会完成（或失败）的操作及其结果值。

```javascript
const p = new Promise((resolve, reject) => {
  // 异步操作
  setTimeout(() => {
    resolve('成功的结果')
  }, 1000)
})
```

## Promise 的三种状态

```
pending（待定）
   ├── resolve(value)  → fulfilled（已成功）
   └── reject(reason)  → rejected（已拒绝）
```

状态一旦改变就**不可逆**：
```javascript
const p = new Promise((resolve, reject) => {
  resolve('第一次')  // 状态变为 fulfilled
  reject('第二次')   // 无效，状态已经 settled
})
p.then(v => console.log(v))  // '第一次'
```

## 基本用法

### then / catch / finally

```javascript
const p = new Promise((resolve, reject) => {
  resolve(42)
})

p
  .then(value => {
    console.log(value)     // 42
    return value * 2       // 返回值传递给下一个 then
  })
  .then(value => {
    console.log(value)     // 84
    throw new Error('出错') // 抛出错误
  })
  .catch(err => {
    console.log(err.message) // '出错'，捕获前面任何一步的错误
  })
  .finally(() => {
    console.log('结束')      // 无论成功失败都会执行
  })
```

### Promise 链式调用的核心规则

`then()` 每次都返回一个**新的 Promise**，取决于回调的返回值：

```javascript
promise.then(value => {
  // 1. 返回普通值 → 新 Promise 立即 resolve(该值)
  return 'hello'

  // 2. 返回 Promise → 新 Promise 跟随该 Promise 的状态
  return fetch('/api/data')

  // 3. 抛出错误 → 新 Promise 立即 reject(该错误)
  throw new Error('fail')

  // 4. 没有返回值 → 新 Promise resolve(undefined)
})
```

## 链式调用 vs 嵌套

```javascript
// ❌ 嵌套 —— 回调地狱
getUser(id, user => {
  getOrders(user.id, orders => {
    getOrderDetail(orders[0].id, detail => {
      console.log(detail)
    })
  })
})

// ✅ 链式调用 —— 扁平化
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetail(orders[0].id))
  .then(detail => console.log(detail))
  .catch(err => console.error(err))  // 统一错误处理
```

## 静态方法

### Promise.all — 全部成功才算成功

```javascript
const p1 = fetch('/api/user')
const p2 = fetch('/api/orders')
const p3 = fetch('/api/products')

Promise.all([p1, p2, p3])
  .then(([user, orders, products]) => {
    // 全部成功，结果按传入顺序排列
  })
  .catch(err => {
    // 任何一个失败就立即 reject
  })
```

### Promise.allSettled — 等所有都结束（ES2020）

```javascript
Promise.allSettled([
  fetch('/api/a'),
  fetch('/api/b'),
  fetch('/api/c')
]).then(results => {
  // results 是数组，每个元素：
  // 成功 → { status: 'fulfilled', value: ... }
  // 失败 → { status: 'rejected', reason: ... }
  results.forEach(r => {
    if (r.status === 'fulfilled') {
      console.log('成功:', r.value)
    } else {
      console.log('失败:', r.reason)
    }
  })
})
```

### Promise.race — 谁先完成用谁

```javascript
// 请求超时控制
const timeout = new Promise((_, reject) => {
  setTimeout(() => reject(new Error('超时')), 5000)
})

Promise.race([fetch('/api/data'), timeout])
  .then(data => console.log(data))
  .catch(err => console.log(err.message)) // 5秒后 → '超时'
```

### Promise.any — 任一成功就算成功（ES2021）

```javascript
Promise.any([
  fetch('https://cdn1.example.com/data'),
  fetch('https://cdn2.example.com/data'),
  fetch('https://cdn3.example.com/data')
])
  .then(response => {
    // 第一个成功的结果
  })
  .catch(err => {
    // 全部失败才 reject
    console.log(err.errors) // 所有错误的数组
  })
```

### 对比表

| 方法 | 成功条件 | 失败条件 | 结果 |
|------|---------|---------|------|
| `Promise.all` | 全部成功 | 任一失败 | 成功值数组 / 第一个错误 |
| `Promise.allSettled` | 永不 reject | 永不 reject | 所有结果数组 |
| `Promise.race` | 第一个完成 | 第一个完成 | 第一个完成的值/原因 |
| `Promise.any` | 任一成功 | 全部失败 | 第一个成功值 / AggregateError |

## async / await

`async/await` 是 Promise 的**语法糖**，让异步代码看起来像同步代码。

### 基本语法

```javascript
// async 函数始终返回一个 Promise
async function getUser() {
  const user = await fetch('/api/user')    // 等待 Promise 完成
  const data = await user.json()           // 等待下一个 Promise
  return data                              // 等价于 Promise.resolve(data)
}

// 调用
getUser().then(data => console.log(data))
```

### await 的本质

`await` 会暂停 async 函数的执行，让出线程给事件循环，等 Promise settle 后恢复执行：

```javascript
async function foo() {
  console.log('1')
  await Promise.resolve()
  console.log('2')         // 微任务，本轮事件循环末尾执行
}
foo()
console.log('3')
// 输出：1 → 3 → 2
```

### 错误处理

```javascript
// 方式一：try...catch
async function getData() {
  try {
    const res = await fetch('/api/data')
    const data = await res.json()
    return data
  } catch (err) {
    console.error('请求失败:', err)
  }
}

// 方式二：catch 链
async function getData() {
  const data = await fetch('/api/data')
    .then(res => res.json())
    .catch(err => console.error(err))
  return data
}

// 方式三：统一包装（utils 风格）
function to(promise) {
  return promise.then(data => [null, data]).catch(err => [err, null])
}

async function getData() {
  const [err, res] = await to(fetch('/api/data'))
  if (err) {
    console.error(err)
    return
  }
  const data = await res.json()
  return data
}
```

### await 的限制

`await` 只能在 `async` 函数内部使用（顶层 await 除外）：

```javascript
// ❌ 报错
const data = await fetch('/api')

// ✅ 顶层 await（ES2022，仅在模块中有效）
// file.mjs
const data = await fetch('/api')
```

## 并发控制

### 串行 vs 并行

```javascript
// ❌ 串行 — 总时间 = t1 + t2 + t3
const a = await fetch('/api/a')  // 1s
const b = await fetch('/api/b')  // 2s
const c = await fetch('/api/c')  // 1s
// 总耗时 4s

// ✅ 并行 — 总时间 = max(t1, t2, t3)
const [a, b, c] = await Promise.all([
  fetch('/api/a'),  // 1s
  fetch('/api/b'),  // 2s
  fetch('/api/c')   // 1s
])
// 总耗时 2s
```

### 限制并发数

```javascript
async function asyncPool(limit, items, iteratorFn) {
  const results = []
  const executing = new Set()

  for (const [index, item] of items.entries()) {
    const p = Promise.resolve().then(() => iteratorFn(item, index))
    results.push(p)
    executing.add(p)

    const clean = () => executing.delete(p)
    p.then(clean, clean)

    if (executing.size >= limit) {
      await Promise.race(executing)
    }
  }

  return Promise.all(results)
}

// 用法：最多同时 3 个请求
await asyncPool(3, urls, url => fetch(url))
```

## 手写 Promise（核心骨架）

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending'
    this.value = undefined
    this.callbacks = []

    const resolve = (value) => {
      if (this.state !== 'pending') return
      this.state = 'fulfilled'
      this.value = value
      this.callbacks.forEach(cb => cb.onFulfilled(value))
    }

    const reject = (reason) => {
      if (this.state !== 'pending') return
      this.state = 'rejected'
      this.value = reason
      this.callbacks.forEach(cb => cb.onRejected(reason))
    }

    try {
      executor(resolve, reject)
    } catch (err) {
      reject(err)
    }
  }

  then(onFulfilled, onRejected) {
    // 处理值穿透
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v
    onRejected = typeof onRejected === 'function' ? onRejected : e => { throw e }

    return new MyPromise((resolve, reject) => {
      const handle = (fn, value) => {
        try {
          const result = fn(value)
          if (result instanceof MyPromise) {
            result.then(resolve, reject)
          } else {
            resolve(result)
          }
        } catch (err) {
          reject(err)
        }
      }

      if (this.state === 'fulfilled') {
        handle(onFulfilled, this.value)
      } else if (this.state === 'rejected') {
        handle(onRejected, this.value)
      } else {
        this.callbacks.push({
          onFulfilled: (v) => handle(onFulfilled, v),
          onRejected: (e) => handle(onRejected, e)
        })
      }
    })
  }

  catch(onRejected) {
    return this.then(null, onRejected)
  }

  finally(callback) {
    return this.then(
      value => MyPromise.resolve(callback()).then(() => value),
      reason => MyPromise.resolve(callback()).then(() => { throw reason })
    )
  }
}
```

## 常见面试问题

### 1. Promise 和回调的区别？

| 回调 | Promise |
|------|---------|
| 控制反转，信任问题 | 链式调用，控制权在自己手里 |
| 嵌套形成回调地狱 | 链式扁平化 |
| 错误处理不统一 | `.catch` 统一捕获 |
| 无法取消 | 可配合 AbortController |

### 2. async/await 和 Promise 的关系？

`async/await` 是 Promise 的语法糖：
- `async` 函数返回 Promise
- `await` 后面跟 Promise，等它 settle 后继续
- 本质上还是微任务，没有脱离 Promise 的机制

### 3. 如何取消一个 Promise？

```javascript
const controller = new AbortController()

fetch('/api/data', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === 'AbortError') {
      console.log('请求被取消')
    }
  })

// 取消
controller.abort()
```

### 4. Promise.all 的错误处理？

```javascript
// ❌ 一个失败全部失败，其他结果丢失
Promise.all([p1, p2, p3]).catch(err => ...)

// ✅ 用 allSettled 拿到所有结果
Promise.allSettled([p1, p2, p3]).then(results => ...)

// ✅ 或者给每个 Promise 加 catch 兜底
Promise.all([
  p1.catch(e => null),
  p2.catch(e => null),
  p3.catch(e => null)
])
```

### 5. then 的第二个参数 vs catch？

```javascript
// 区别一：catch 能捕获 onFulfilled 中的错误
promise.then(
  value => { throw new Error('oops') },  // 这里的错误
  err => console.log(err)                 // 第二个参数捕获不到
)
promise
  .then(value => { throw new Error('oops') })
  .catch(err => console.log(err))          // catch 能捕获 ✅

// 区别二：catch 只能有一个，then 可以链式多个
promise
  .then(v => v * 2)
  .then(v => v * 3)
  .catch(err => ...)  // 捕获前面所有 then 的错误
```

### 6. 微任务队列中的执行顺序？

```javascript
Promise.resolve()
  .then(() => console.log(1))
  .then(() => console.log(2))

Promise.resolve()
  .then(() => console.log(3))
  .then(() => console.log(4))

// 输出：1 → 3 → 2 → 4
// 原因：每个 then 创建新的微任务，按入队顺序执行
```

### 7. async 函数 return 和 throw 会怎样？

```javascript
async function foo() {
  return 42           // 等价于 Promise.resolve(42)
}
// 等价于
async function bar() {
  throw new Error()   // 等价于 Promise.reject(new Error())
}
```

## 核心要点

- Promise 三种状态：pending → fulfilled / rejected，**不可逆**
- `then()` 返回新 Promise，值取决于回调返回值
- `Promise.all` 并发执行，任一失败即 reject；`allSettled` 等全部结束
- `async/await` 是语法糖，本质还是微任务
- `await` 暂停函数执行，让出线程给事件循环
- 并行请求用 `Promise.all`，串行用 `await` 逐个等待
- 错误处理：`try...catch` 或 `.catch` 链
