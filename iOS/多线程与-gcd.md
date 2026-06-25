---
type: review
---
# 多线程与 GCD

> **iOS 有哪些多线程方案？GCD 的串行队列、并发队列、主队列分别是什么？**`dispatch_sync` **和** `dispatch_async` **有什么区别？有哪些常见会导致死锁的场景？**

---

## 一、iOS 多线程方案总览

| 方案 | 特点 | 使用场景 |
|------|------|---------|
| **Pthreads** | POSIX 标准 C API，跨平台，需手动管理线程生命周期 | 几乎不在 iOS 直接使用 |
| **NSThread** | 面向对象封装，可手动管理线程 | 需要精细控制线程（如设置线程名、优先级） |
| **GCD** | 基于队列的并发模型，C API，系统管理线程池 | **最常用**，绝大多数并发场景 |
| **NSOperation / NSOperationQueue** | 基于 GCD 的高级封装，支持依赖、取消、优先级 | 复杂任务编排（如依赖关系、最大并发数） |

---

## 二、GCD 核心概念

### 2.1 队列类型

```text
┌─────────────────────────────────────────────────┐
│                  Dispatch Queues                 │
├─────────────┬───────────────┬───────────────────┤
│ Serial      │ Concurrent    │ Main              │
│ (串行队列)   │ (并发队列)     │ (主队列)          │
├─────────────┼───────────────┼───────────────────┤
│ 一次执行一个  │ 多个任务同时    │ 特殊串行队列       │
│ 任务         │ 执行          │ 在主线程执行        │
│ FIFO 顺序    │ FIFO 开始顺序  │ 更新 UI 必须用它   │
└─────────────┴───────────────┴───────────────────┘
```

**串行队列（Serial Queue）**
- 一次只执行一个任务，按 FIFO 顺序
- 用于同步共享资源，避免数据竞争
- 创建：`dispatch_queue_create("com.example.serial", DISPATCH_QUEUE_SERIAL)`

**并发队列（Concurrent Queue）**
- 多个任务可以同时执行
- 系统提供四个全局并发队列（QoS）：`QOS_CLASS_USER_INTERACTIVE`、`QOS_CLASS_USER_INITIATED`、`QOS_CLASS_UTILITY`、`QOS_CLASS_BACKGROUND`
- 获取：`dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0)`

**主队列（Main Queue）**
- 全局唯一的串行队列，绑定主线程
- 用于 UI 更新、接收用户交互事件
- 获取：`dispatch_get_main_queue()`

### 2.2 QoS（Quality of Service）优先级

| QoS 级别 | 用途 | 系统优先级 |
|----------|------|----------|
| `.userInteractive` | 动画、事件处理、UI 更新 | 最高 |
| `.userInitiated` | 用户触发的需要立即结果的任务 | 高 |
| `.default` | 默认（介于 userInitiated 和 utility 之间） | 中 |
| `.utility` | 长时间任务（下载、导入） | 低 |
| `.background` | 后台同步、备份、索引 | 最低 |

---

## 三、dispatch_sync vs dispatch_async

```objectivec
// 同步派发：阻塞当前线程，等任务执行完才返回
dispatch_sync(queue, ^{
    // 任务代码
});

// 异步派发：不阻塞当前线程，立即返回
dispatch_async(queue, ^{
    // 任务代码
});
```

| 特性 | `dispatch_sync` | `dispatch_async` |
|------|----------------|-----------------|
| 是否阻塞当前线程 | ✅ 阻塞，等任务完成 | ❌ 不阻塞，立即返回 |
| 是否具备开新线程的能力 | ❌ 不会开新线程 | ✅ 可能开新线程（并发队列） |
| 返回时机 | 任务执行完后返回 | 任务入队后立即返回 |
| 常见用途 | 获取并发队列结果、初始化共享资源 | 耗时任务、异步回调 |

### 关键理解

- **sync + 串行队列**：在当前线程顺序执行（不开新线程）
- **sync + 并发队列**：在当前线程顺序执行（不开新线程）
- **async + 串行队列**：开一个新线程，顺序执行
- **async + 并发队列**：开多个新线程，并发执行

---

## 四、GCD 常用 API

### 4.1 dispatch_once — 单例

```objectivec
+ (instancetype)sharedInstance {
    static MyClass *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[MyClass alloc] init];
    });
    return instance;
}
```

> 线程安全，内部通过原子操作 + 屏障保证只执行一次。

### 4.2 dispatch_after — 延迟执行

```objectivec
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
               dispatch_get_main_queue(), ^{
    // 2 秒后执行（注意：不是精确的定时器，是"至少 2 秒后"）
});
```

### 4.3 dispatch_group — 任务组

```objectivec
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
    // 任务 1
});

dispatch_group_async(group, dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
    // 任务 2
});

// 方式 A：阻塞等待
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);

// 方式 B：通知回调（推荐，不阻塞）
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    // 所有任务完成，回到主线程
});
```

### 4.4 dispatch_barrier_async — 栅栏（读写锁）

```objectivec
dispatch_queue_t queue = dispatch_queue_create("com.example.rwqueue",
                                                DISPATCH_QUEUE_CONCURRENT);

// 读操作 — 并发
dispatch_async(queue, ^{ /* 读 */ });
dispatch_async(queue, ^{ /* 读 */ });

// 写操作 — 栅栏，独占访问
dispatch_barrier_async(queue, ^{ /* 写 */ });

// 读操作 — 写完后并发
dispatch_async(queue, ^{ /* 读 */ });
```

> 栅栏任务之前的所有任务执行完 → 栅栏任务独占执行 → 栅栏之后的任务继续并发。

### 4.5 dispatch_semaphore — 信号量

```objectivec
dispatch_semaphore_t sem = dispatch_semaphore_create(1); // 初始值 1

dispatch_async(queue, ^{
    dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER); // -1，小于 0 则等待
    // 临界区
    dispatch_semaphore_signal(sem); // +1，唤醒等待的线程
});
```

> 信号量初始值为 1 时，等价于互斥锁（mutex）。初始值为 N 时，最多允许 N 个任务同时执行。

### 4.6 dispatch_apply — 并发循环

```objectivec
dispatch_apply(10, dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^(size_t index) {
    // 会并发执行 10 次，但会阻塞当前线程直到全部完成
    NSLog(@"%zu", index);
});
```

---

## 五、死锁场景详解

### 5.1 经典死锁：sync 到当前串行队列

```objectivec
// ❌ 死锁！
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"这行永远不会执行");
});
```

**逐步分析：**

```text
主线程正在执行代码
    ↓
dispatch_sync 说：把任务放到主队列，等它执行完
    ↓
但主队列是串行的，当前任务还没执行完（在等 sync 返回）
    ↓
放进去的任务排在当前任务后面，永远等不到
    ↓
💀 死锁
```

**原因**：`dispatch_sync` 要求任务执行完才返回，而任务在主队列，需要主线程执行。但主线程被 `sync` 阻塞了 → 相互等待 → 死锁。

**规律**：**在当前队列上调用 `dispatch_sync` 派发任务到同一个队列**，必死锁。

### 5.2 嵌套 sync 导致死锁

```objectivec
dispatch_queue_t queue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);

dispatch_async(queue, ^{
    NSLog(@"1");  // 新线程执行

    // ❌ 死锁！
    dispatch_sync(queue, ^{
        NSLog(@"2");  // 永远不会执行
    });

    NSLog(@"3");  // 永远不会执行
});
```

**逐步分析：**

```text
线程 A 正在执行 block A（在串行队列中）
    ↓
dispatch_sync 把 block B 放到同一个串行队列
    ↓
串行队列要等 block A 执行完才能执行 block B
    ↓
但 block A 在等 dispatch_sync 返回（即等 block B 执行完）
    ↓
A 等 B，B 等 A → 💀 死锁
```

**关键点**：`dispatch_async` 开了新线程执行 block，但 block 内部又 `dispatch_sync` 到**同一个串行队列**，形成自己等自己的循环。

### 5.3 互相 sync 等待

```objectivec
// ❌ 两个线程互相 sync 等待对方
dispatch_queue_t queueA = dispatch_queue_create("com.example.a", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t queueB = dispatch_queue_create("com.example.b", DISPATCH_QUEUE_SERIAL);

dispatch_async(queueA, ^{
    dispatch_sync(queueB, ^{  // 等 queueB 执行
        // ...
    });
});

dispatch_async(queueB, ^{
    dispatch_sync(queueA, ^{  // 等 queueA 执行
        // ...
    });
});
```

**逐步分析：**

```text
线程 A：async 到 queueA，开始执行 block A
    ↓
block A 调用 dispatch_sync(queueB, block B)
    ↓
sync 要求 block B 在 queueB 上执行完才返回
    ↓
queueB 当前线程 B 正在执行 block B（还没执行完）
    ↓
block B 调用 dispatch_sync(queueA, block C)
    ↓
sync 要求 block C 在 queueA 上执行完才返回
    ↓
但 queueA 正在执行 block A，block A 在等 queueB 的 sync 返回
    ↓
block A 等 block B → block B 等 block C → block C 等 block A 完成
    ↓
三方循环等待 → 💀 死锁
```

**本质**：两个（或多个）线程通过 `dispatch_sync` 互相等待对方队列的任务完成，形成循环依赖。与线程间互相持锁的经典死锁本质相同。

### 5.4 安全的写法示例

```objectivec
// ✅ async 到主队列 — 不会死锁
dispatch_async(dispatch_get_main_queue(), ^{
    // 安全
});

// ✅ sync 到其他队列 — 不会死锁
dispatch_queue_t otherQueue = dispatch_queue_create("com.example.other", DISPATCH_QUEUE_SERIAL);
dispatch_sync(otherQueue, ^{
    // 安全，因为 otherQueue 不是当前队列
});

// ✅ sync 到全局并发队列 — 不会死锁
dispatch_sync(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
    // 安全
});

// ✅ 不会死锁！sync 到不同的串行队列
dispatch_sync(queueA, ^{
    dispatch_sync(queueB, ^{  // queueB ≠ queueA
        NSLog(@"OK");
    });
});

// ✅ 不会死锁！在主线程 sync 到自定义串行队列
dispatch_sync(mySerialQueue, ^{
    NSLog(@"OK");  // 在主线程执行，不涉及主队列
});

// ✅ 不会死锁！并发队列 sync 到自身
dispatch_queue_t concurrent = dispatch_queue_create("c", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrent, ^{
    dispatch_sync(concurrent, ^{
        NSLog(@"OK");  // 并发队列可以同时执行多个任务
    });
});
```

### 5.5 死锁判断规则

```
判断是否死锁的核心问题：
1. 当前在哪个队列执行？（设为 Q_current）
2. 要 sync 派发到哪个队列？（设为 Q_target）
3. Q_target 是否是串行队列？
4. Q_current == Q_target ？

如果 Q_current == Q_target 且是串行队列 + sync 派发 → 死锁 💀
```

**用规则逐一验证上面的死锁场景：**

---

**5.1 经典死锁：sync 到主队列**

```objectivec
// 在主线程执行
dispatch_sync(dispatch_get_main_queue(), ^{ ... });
```

| 步骤 | 分析 |
|------|------|
| ① Q_current | 主队列（主线程正在执行） |
| ② Q_target | `dispatch_get_main_queue()` = 主队列 |
| ③ 是否串行？ | ✅ 主队列是串行队列 |
| ④ Q_current == Q_target？ | ✅ 主队列 == 主队列 |

**结论**：串行队列 + sync 派发到自身 → 💀 **死锁**

---

**5.2 嵌套 sync 到同一串行队列**

```objectivec
dispatch_async(queue, ^{           // queue 是自定义串行队列
    dispatch_sync(queue, ^{ ... }); // sync 到同一个 queue
});
```

| 步骤 | 分析 |
|------|------|
| ① Q_current | `queue`（async 开的线程正在这个串行队列上执行 block A） |
| ② Q_target | `queue`（sync 派发的目标也是同一个 queue） |
| ③ 是否串行？ | ✅ `DISPATCH_QUEUE_SERIAL` |
| ④ Q_current == Q_target？ | ✅ 同一个 queue |

**结论**：串行队列 + sync 派发到自身 → 💀 **死锁**

---

**5.3 互相 sync 等待（逐队列分析）**

```objectivec
dispatch_async(queueA, ^{
    dispatch_sync(queueB, ^{ ... });  // 在 queueA 上 sync 到 queueB
});
dispatch_async(queueB, ^{
    dispatch_sync(queueA, ^{ ... });  // 在 queueB 上 sync 到 queueA
});
```

**单独看 sync(queueB) 这一步：**

| 步骤 | 分析 |
|------|------|
| ① Q_current | `queueA` |
| ② Q_target | `queueB` |
| ③ 是否串行？ | ✅ 两个都是串行队列 |
| ④ Q_current == Q_target？ | ❌ queueA ≠ queueB |

单看这一步，**不会死锁**。

**单独看 sync(queueA) 这一步：**

| 步骤 | 分析 |
|------|------|
| ① Q_current | `queueB` |
| ② Q_target | `queueA` |
| ③ 是否串行？ | ✅ 两个都是串行队列 |
| ④ Q_current == Q_target？ | ❌ queueB ≠ queueA |

单看这一步，也**不会死锁**。

**那为什么还是会死锁？**

因为上面的规则只适用于**单个队列自身的循环等待**。5.3 的情况是**跨队列的循环等待**，需要扩展判断：

```
扩展判断：多个 sync 调用之间是否形成环形依赖？

线程 A：在 queueA 上 sync 到 queueB → 等 queueB 执行完
线程 B：在 queueB 上 sync 到 queueA → 等 queueA 执行完

依赖链：queueA 等 queueB → queueB 等 queueA → 回到起点 → 环形依赖
```

**结论**：虽然单次 sync 不满足「Q_current == Q_target」，但两个 sync 调用形成了 **A 等 B、B 等 A 的环形依赖** → 💀 **死锁**

---

**总结：两层死锁判断**

```text
第一层（单次 sync）：
  Q_current == Q_target 且是串行队列 → 必死锁
  适用：5.1、5.2

第二层（多次 sync 之间的关系）：
  检查是否存在环形依赖：A 等 B → B 等 C → ... → 等 A
  适用：5.3 互相 sync 等待

核心本质相同：都是「循环等待」——要么自己等自己，要么互相等对方。
```

---

## 六、NSOperation vs GCD

| 特性 | GCD | NSOperation |
|------|-----|-------------|
| 抽象层级 | C API，底层 | ObjC 对象，高层 |
| 取消任务 | ❌ 不支持（只能用 flag 标记） | ✅ `cancel` 方法 |
| 依赖关系 | ❌ 需要手动实现 | ✅ `addDependency:` |
| 最大并发数 | 由系统控制 | `maxConcurrentOperationCount` |
| 任务状态 | 无状态 | `isReady` / `isExecuting` / `isFinished` |
| KVO | 不支持 | 支持状态 KVO |
| 暂停/恢复 | 不支持 | `queue.isSuspended = YES` |

```objectivec
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
queue.maxConcurrentOperationCount = 3;

NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{ /* 任务1 */ }];
NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{ /* 任务2 */ }];

[op2 addDependency:op1]; // op2 依赖 op1，op1 完成后才执行 op2

[queue addOperation:op1];
[queue addOperation:op2];
```

---

## 七、Swift 并发：从 GCD 到 async/await

### 7.1 传统 GCD 写法（Swift）

```swift
DispatchQueue.global(qos: .userInitiated).async {
    let result = self.fetchData()
    DispatchQueue.main.async {
        self.updateUI(result)
    }
}
```

### 7.2 async/await（iOS 15+）

```swift
func loadData() async {
    let result = await fetchData()  // 自动切到后台
    updateUI(result)                // 自动回到主线程（如果 Actor 是 @MainActor）
}
```

### 7.3 Task 和 TaskGroup

```swift
// 单个异步任务
Task {
    let data = await fetchFromServer()
    // 自动在 MainActor 更新 UI
}

// 并发多个任务
await withTaskGroup(of: Data.self) { group in
    group.addTask { await self.fetch(url1) }
    group.addTask { await self.fetch(url2) }
    for await result in group {
        // 收集结果
    }
}
```

### 7.4 Actor — 数据竞争防护

```swift
actor BankAccount {
    var balance: Double = 0
    
    func deposit(_ amount: Double) {
        balance += amount  // 自动串行化，无需手动加锁
    }
}
```

> Actor 保证对可变状态的访问是串行的，编译器层面防止数据竞争。

---

## 八、总结速查表

| 概念 | 关键点 |
|------|--------|
| GCD 队列类型 | Serial（串行）、Concurrent（并发）、Main（主队列） |
| sync vs async | sync 阻塞当前线程；async 不阻塞 |
| dispatch_once | 线程安全单例，原子操作保证只执行一次 |
| dispatch_group | 等待多个并发任务全部完成 |
| dispatch_barrier | 读写锁：读并发，写独占 |
| dispatch_semaphore | 信号量，初始值 1 时 = 互斥锁 |
| 死锁核心 | **串行队列 + sync 派发到自身 = 死锁** |
| NSOperation 优势 | 支持取消、依赖、最大并发数、状态管理 |
| Swift async/await | 替代 GCD 回调地狱，编译器保证线程安全 |
| Actor | Swift 并发的并发原语，自动防止数据竞争 |
