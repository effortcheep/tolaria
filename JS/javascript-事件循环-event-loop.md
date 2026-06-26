---
type: review
---
# JavaScript 事件循环（Event Loop）

## 什么是事件循环

事件循环是 JavaScript 实现**单线程异步**的核心机制。JS 引擎本身是单线程的，但通过事件循环 + 任务队列，能够非阻塞地处理 I/O、定时器、网络请求等异步操作。

## 为什么需要事件循环

JavaScript 是单线程语言（主线程只有一个），如果所有操作都同步执行，遇到网络请求、定时器等耗时操作时页面会"卡死"。事件循环解决了这个问题：将耗时操作交给宿主环境（浏览器/Node.js）处理，主线程继续执行，完成后通过回调通知。

## 核心概念

### 1. 调用栈（Call Stack）

同步代码的执行栈。函数调用时压栈，执行完毕弹栈。栈为空时才会从任务队列取任务。

```js
function foo() {
  console.log('foo');
  bar();
}
function bar() {
  console.log('bar');
}
foo();
// 调用栈: main → foo → bar → pop bar → pop foo → pop main
```

### 2. 宏任务（Macro Task）

每次事件循环**从队列取一个**宏任务执行。

常见的宏任务：
- `setTimeout` / `setInterval`
- `setImmediate`（Node.js）
- I/O 回调
- UI 渲染（浏览器）
- `requestAnimationFrame`（浏览器，严格来说在渲染前）
- script 整体代码

### 3. 微任务（Micro Task）

当前宏任务执行完后，**清空所有微任务**再执行下一个宏任务。

常见的微任务：
- `Promise.then` / `.catch` / `.finally`
- `MutationObserver`（浏览器）
- `queueMicrotask()`
- `process.nextTick`（Node.js，优先级最高）
- `await` 后面的代码（等价于 `.then`）

## 执行顺序

```
1. 执行同步代码（整段 script 作为第一个宏任务）
2. 清空所有微任务队列（包括微任务中产生的新微任务）
3. 渲染（浏览器，如果需要）
4. 取下一个宏任务执行
5. 重复步骤 2-4
```

### 为什么微任务优先？

这不是一个"规定"，而是有明确的设计动机和技术原因：

**1. 设计动机：保证数据一致性**

微任务（如 Promise 回调）通常是对**当前操作**的延续。比如一次网络请求完成后，你可能需要更新状态、触发 UI 变化、发起下一个请求。如果这些"后续处理"被推迟到下一个宏任务，而中间插进来一个渲染或另一个宏任务，用户就可能看到**中间态**——数据变了但 UI 没跟上，或者 UI 变了但数据还没准备好。

微任务优先确保了一个完整操作的所有步骤**连续执行、不被打断**。

考虑这个场景：

```js
console.log('start')

setTimeout(() => {
  console.log('timeout')
}, 0)

Promise.resolve()
  .then(() => {
    console.log('promise1')
  })
  .then(() => {
    console.log('promise2')  // 依赖 promise1 的结果
  })

console.log('end')
```

**输出：`start → end → promise1 → promise2 → timeout`**

如果微任务不优先，`timeout` 就会插到 `promise1` 和 `promise2` 之间。假设 `promise1` 修改了某个共享状态，`promise2` 读取该状态来做后续逻辑，而 `timeout` 回调也修改了这个状态——那 `promise2` 读到的就是被 timeout 篡改过的脏数据，逻辑就乱了。

**微任务链作为一个整体原子执行完，中间不被宏任务打断**，这才保证了 Promise 链内部的状态一致性。

**2. 技术原因：微任务是"同步代码的延伸"**

从代码逻辑上看，Promise 链和 async/await 本质上是**同步代码的异步化写法**：

```js
// 这段代码的逻辑意图是连续的
const data = await fetchData();
const result = processData(data);
updateUI(result);
```

虽然 `await` 后面的代码变成了微任务，但程序员的**逻辑意图**是它们应该紧跟着执行，中间不应该插入无关操作。微任务优先就是为了让这种"逻辑连续性"得以保持。

**3. 对比宏任务**

宏任务代表的是**独立的、外部的事件**——一个定时器到期、一次网络响应到达、一次用户点击。这些事件之间没有逻辑依赖关系，所以它们之间可以安全地插入渲染、插入其他宏任务。

微任务代表的是**当前操作的内部延续**——一个 Promise 的 then、一个 MutationObserver 的回调。它们和当前操作紧密耦合，必须尽快处理完才能保证状态一致。

**4. 类比**

```
宏任务 = 不同客户的需求（互相独立，可以排队）
微任务 = 当前客户还没交代完的事（必须当场处理完）
```

你在接待客户 A 时，A 又交代了两件事（产生了微任务），你会先把 A 的事全部处理完，再去叫下一个客户 B。这就是微任务优先的逻辑。

**5. 规范层面**

HTML 规范在定义事件循环时，明确要求在每个宏任务之后、渲染之前，执行"清空微任务队列"步骤（perform a microtask checkpoint）。这不是实现细节，而是**规范强制要求**的，所有符合规范的 JS 引擎都必须遵守。

## 经典示例

### 示例 1：基础

```js
console.log('1');

setTimeout(() => {
  console.log('2');
}, 0);

Promise.resolve().then(() => {
  console.log('3');
});

console.log('4');
```

**输出：`1 → 4 → 3 → 2`**

解析：
1. 同步：打印 `1`、`4`
2. 微任务：Promise.then → 打印 `3`
3. 宏任务：setTimeout → 打印 `2`

### 示例 2：微任务中产生微任务

```js
Promise.resolve().then(() => {
  console.log('micro1');
  Promise.resolve().then(() => {
    console.log('micro2');
  });
}).then(() => {
  console.log('micro3');
});
```

**输出：`micro1 → micro2 → micro3`**

解析：微任务队列会被**完全清空**（包括执行过程中新产生的微任务），然后才进入下一个宏任务。第一个 `.then` 执行时产生新的微任务 `micro2`，第二个 `.then`（`micro3`）依赖第一个 `.then` 完成后才入队。

### 示例 3：async/await

```js
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');
setTimeout(() => {
  console.log('setTimeout');
}, 0);

async1();

new Promise((resolve) => {
  console.log('promise1');
  resolve();
}).then(() => {
  console.log('promise2');
});

console.log('script end');
```

**输出：**
```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

解析：
- `await async2()` 等价于 `async2().then(() => { console.log('async1 end') })`
- `async1 end` 和 `promise2` 都是微任务，按入队顺序执行

### 示例 4：综合复杂题

```js
console.log('1');

setTimeout(() => {
  console.log('2');
  Promise.resolve().then(() => console.log('3'));
}, 0);

Promise.resolve().then(() => {
  console.log('4');
  setTimeout(() => console.log('5'), 0);
});

Promise.resolve().then(() => console.log('6'));

console.log('7');
```

**输出：`1 → 7 → 4 → 6 → 2 → 3 → 5`**

## 浏览器 vs Node.js 的区别

### Node.js 事件循环（libuv）

Node.js 的事件循环更复杂，分为**6 个阶段**：

```
   ┌───────────────────────────┐
┌─>│         timers            │  ← setTimeout / setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← 系统级回调（TCP 错误等）
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← 内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │          poll             │  ← I/O 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │          check            │  ← setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  ← socket.on('close')
│  └─────────────┬─────────────┘
│                │
└────────────────┘
```

**关键区别：**
- `process.nextTick` 优先级**高于**所有微任务，在每个阶段切换前执行
- `setImmediate` 在 check 阶段执行，`setTimeout` 在 timers 阶段执行
- 在 I/O 回调中，`setImmediate` 一定先于 `setTimeout` 执行

```js
// Node.js
setImmediate(() => console.log('immediate'));
setTimeout(() => console.log('timeout'), 0);
// 输出顺序不确定（取决于进入事件循环的时机）
```

```js
const fs = require('fs');
fs.readFile(__filename, () => {
  setImmediate(() => console.log('immediate'));
  setTimeout(() => console.log('timeout'), 0);
});
// 一定输出: immediate → timeout（因为在 I/O 回调中）
```

### 微任务执行时机的差异

```js
// Node.js 11+ 行为（与浏览器一致：每个宏任务后清空微任务）
setTimeout(() => {
  console.log('timeout1');
  Promise.resolve().then(() => console.log('promise1'));
}, 0);
setTimeout(() => {
  console.log('timeout2');
  Promise.resolve().then(() => console.log('promise2'));
}, 0);
// Node.js 11+: timeout1 → promise1 → timeout2 → promise2
// Node.js 10:  timeout1 → timeout2 → promise1 → promise2
```

## requestAnimationFrame

`requestAnimationFrame`（rAF）不是宏任务也不是微任务，它在**浏览器渲染前**执行：

```
宏任务 → 微任务 → rAF → 渲染 → rIC（requestIdleCallback）
```

```js
requestAnimationFrame(() => {
  console.log('rAF');
});
setTimeout(() => {
  console.log('timeout');
}, 0);
// rAF 可能先于 timeout（如果下一帧即将到来）
```

## 常见面试问题

### Q1：为什么 `setTimeout(fn, 0)` 不是立即执行？

答：`setTimeout` 的最小延迟是 4ms（浏览器规范），且它进入宏任务队列，必须等当前宏任务和所有微任务执行完才轮到它。

### Q2：Promise 的执行顺序为什么比 setTimeout 先？

答：Promise.then 是微任务，微任务在当前宏任务结束后立即清空；setTimeout 是宏任务，要等下一个事件循环周期。

### Q3：async/await 的本质是什么？

答：`await` 是 `Promise.then` 的语法糖。`await` 后面的代码会被包装成微任务。`async` 函数返回 Promise。

### Q4：如何实现一个 `nextTick`（微任务）？

```js
function nextTick(fn) {
  if (typeof queueMicrotask === 'function') {
    queueMicrotask(fn);
  } else if (typeof MutationObserver !== 'undefined') {
    const observer = new MutationObserver(fn);
    const node = document.createTextNode('');
    observer.observe(node, { characterData: true });
    node.data = '1';
  } else {
    Promise.resolve().then(fn);
  }
}
```

### Q5：Vue 的 `nextTick` 为什么用微任务？

Vue 的 DOM 更新是异步批量的，`nextTick` 确保在 DOM 更新完成后执行回调。使用微任务可以在当前宏任务结束后、浏览器渲染前拿到更新后的 DOM。

### Q6：事件循环中的"饥饿"问题是什么？

答：如果微任务中不断产生新的微任务，会阻塞宏任务的执行（如 setTimeout 回调、UI 渲染），导致页面卡死。解决方案：将长任务拆分为多个宏任务。

```js
// 错误：微任务可能饿死宏任务
function processLargeArray(arr) {
  return new Promise((resolve) => {
    arr.forEach(item => {
      processItem(item);
    });
    resolve();
  });
}

// 正确：用 setTimeout 拆分
function processLargeArray(arr, chunk = 100) {
  let i = 0;
  function processChunk() {
    const end = Math.min(i + chunk, arr.length);
    for (; i < end; i++) {
      processItem(arr[i]);
    }
    if (i < arr.length) {
      setTimeout(processChunk, 0);
    }
  }
  processChunk();
}
```

## 总结

| 概念 | 说明 |
|------|------|
| 调用栈 | 同步代码执行的地方 |
| 宏任务 | 事件循环每次取一个执行 |
| 微任务 | 当前宏任务结束后全部清空 |
| 执行顺序 | 同步 → 微任务 → 渲染 → 宏任务 |
| `await` | 等价于 `.then`，后面的代码是微任务 |
| Node.js | 6 阶段循环，`process.nextTick` 优先级最高 |
| rAF | 渲染前执行，非宏非微 |
