---
type: review
related_to: "[[iOS/ios-应用的完整启动流程]]"
---
# RunLoop

> **RunLoop 是什么？它和线程是什么关系？RunLoop 的 Mode 有哪些？实际开发中哪些场景用到了 RunLoop？**

---

## 一、RunLoop 是什么？

RunLoop 是一个**事件循环（Event Loop）**，它的核心逻辑本质上是一个 `do-while` 循环：

```
while (1) {
    // 1. 通知 Observer：即将处理 Timer
    // 2. 通知 Observer：即将处理 Source0
    // 3. 处理 Source0
    // 4. 如果有 Source1，跳到第 9 步
    // 5. 通知 Observer：线程即将休眠
    // 6. 线程休眠，等待唤醒事件：
    //    - 端口事件（Source1）
    //    - Timer 到时
    //    - RunLoop 超时
    //    - 被手动唤醒
    // 7. 通知 Observer：线程被唤醒
    // 8. 处理唤醒事件，回到第 2 步
    // 9. 处理 Source1，回到第 2 步
}
```

**RunLoop 的作用**：保持线程不退出，在没有事件处理时让线程休眠（不占用 CPU），有事件时被唤醒处理事件。

---

## 二、RunLoop 和线程的关系

| 关系 | 说明 |
|------|------|
| **一一对应** | 每个线程都有且仅有一个 RunLoop（懒加载创建） |
| **主线程 RunLoop** | 系统自动创建并启动，所以 App 启动后不会立即退出 |
| **子线程 RunLoop** | 需要手动获取并启动，否则子线程执行完任务就销毁了 |
| **不需自己创建** | 只能获取（`CFRunLoopGetMain()` / `CFRunLoopGetCurrent()`），不能手动 `new` |
| **线程销毁 = RunLoop 销毁** | 线程结束时，其 RunLoop 也随之销毁 |

**存储方式**：RunLoop 对象存储在一个全局的 `Dictionary` 中，key 是线程的 `pthread_t`，value 是 `CFRunLoopRef`。这是一个线程安全的字典。

---

## 三、RunLoop 的数据结构（CFRunLoop）

```
CFRunLoop
├── _currentMode: CFRunLoopMode        // 当前运行的 Mode
├── _modes: CFMutableSet<CFRunLoopMode> // 所有 Mode 的集合
├── _commonModes: CFMutableSet<CFStringRef> // 标记为 "common" 的 Mode 集合
└── _commonModeItems: ...               // 在所有 common Mode 中共享的 Source/Timer/Observer

CFRunLoopMode
├── _name: CFStringRef                  // Mode 名称
├── _sources0: CFMutableSet             // 需要手动触发的事件源（触摸事件、performSelector:onThread:）
├── _sources1: CFMutableSet             // 基于 Port 的事件源（系统事件、mach port）
├── _timers: CFMutableArray             // 定时器集合
└── _observers: CFMutableArray          // 观察者集合（监听 RunLoop 状态变化）
```

### Source0 vs Source1

| 类型 | 特点 | 举例 |
|------|------|------|
| **Source0** | 非基于 Port，需要手动标记为待处理并唤醒 RunLoop | 触摸事件、`performSelector:onThread:` |
| **Source1** | 基于 Port（mach port），系统内核直接唤醒 RunLoop | 系统事件、`NSPort` 通信 |

### Observer 监听的状态

```c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 进入RunLoop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 被唤醒
    kCFRunLoopExit          = (1UL << 7), // 退出RunLoop
    kCFRunLoopAllActivities = 0x0FFFFFFFU // 所有状态
};
```

---

## 四、RunLoop 的 Mode

### 系统提供的 Mode

| Mode | 名称 | 说明 |
|------|------|------|
| **DefaultMode** | `NSDefaultRunLoopMode` | 默认模式，App 空闲时处于此模式。大部分事件在此处理 |
| **UITrackingMode** | `UITrackingRunLoopMode` | 滑动追踪模式，ScrollView 滑动时进入此模式 |
| **CommonModes** | `NSRunLoopCommonModes` | 不是一个真正的 Mode，而是一个 **Mode 集合**（包含 Default + Tracking） |
| **InitilizationMode** | `UIInitializationRunLoopMode` | App 启动时使用，启动完成后不再使用 |
| **SystemMode** | `GSEventReceiveRunLoopMode` | 接收系统内部事件，通常用不到 |

### Mode 的核心机制

**一次 RunLoop 只能运行在一个 Mode 下**。切换 Mode 时，必须先退出当前 Loop，再以新 Mode 重新进入。

这个设计的目的是：**隔离不同类型的事件源**，避免相互干扰。比如滑动 ScrollView 时切到 UITrackingMode，Timer 在 DefaultMode 下就不会被触发，保证滑动流畅。

### CommonMode 的特殊性

`NSRunLoopCommonModes` 不是独立 Mode，而是多个 Mode 的集合。将 Source/Timer 添加到 CommonMode = 同时添加到集合中的每个 Mode。

默认包含：`NSDefaultRunLoopMode` + `UITrackingRunLoopMode`

```objc
// Timer 在滑动时也触发的解决方案
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

---

## 五、RunLoop 的启动流程

### CFRunLoopRun 核心流程

```
CFRunLoopRun()
  → CFRunLoopRunSpecific(runloop, currentMode, ...)
      → 通知 Observer: 即将进入 RunLoop (__CFRunLoopRun 开始)
      → 通知 Observer: 即将处理 Timer
      → 通知 Observer: 即将处理 Source0
      → 处理 Source0 (CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION)
      → 如果有 Source1 → 跳到处理 Source1
      → 通知 Observer: 即将休眠
      → mach_msg() 系统调用，线程休眠
      → 被唤醒
      → 通知 Observer: 已被唤醒
      → 处理唤醒事件（Timer / Source1 / 超时 / 被手动唤醒）
      → 回到循环顶部
```

### RunLoop 退出条件

1. Mode 中没有任何 Source/Timer/Observer（立即退出）
2. 超时（`CFRunLoopRun` 有超时参数）
3. 被手动调用 `CFRunLoopStop()` 停止

---

## 六、实际开发中的 RunLoop 应用场景

### 1. NSTimer 在滑动时不工作

**问题**：`[NSTimer scheduledTimerWithTimeInterval:...]` 默认添加到 `NSDefaultRunLoopMode`，滑动时 RunLoop 切到 `UITrackingRunLoopMode`，Timer 就不触发了。

**解决**：
```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(tick) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

### 2. 子线程中使用 NSTimer

**问题**：子线程默认没有启动 RunLoop，Timer 无法工作。

**解决**：
```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(tick) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run]; // 启动 RunLoop（注意：这会永久运行）
});
```

### 3. PerformSelector:onThread: 线程间通信

```objc
// 在指定线程执行方法
[self performSelector:@selector(doWork) onThread:targetThread withObject:nil waitUntilDone:NO];
```

这实际上是往目标线程的 RunLoop 中添加了一个 **Source0** 事件。如果目标线程的 RunLoop 没有启动，方法不会被执行。

### 4. 监控卡顿（卡顿检测方案）

利用 Observer 监听 RunLoop 状态，如果从 `BeforeSources` / `BeforeWaiting` 到下一次状态变化的时间过长（如 > 16ms × N），说明主线程卡顿。

```swift
// 卡顿检测核心思路
let observer = CFRunLoopObserverCreateWithHandler(
    kCFAllocatorDefault,
    CFRunLoopActivity.allActivities.rawValue,
    true, // 重复监听
    0     // 优先级
) { observer, activity in
    switch activity {
    case .entry:        // 进入 RunLoop
    case .beforeTimers: // 即将处理 Timer
    case .beforeSources: // 即将处理 Source ← 记录时间戳
    case .beforeWaiting: // 即将休眠 ← 计算耗时
    case .afterWaiting:  // 被唤醒
    case .exit:          // 退出
    default: break
    }
}
CFRunLoopAddObserver(CFRunLoopGetMain(), observer, CFRunLoopMode.defaultMode)
```

### 5. AutoreleasePool 的管理

App 启动后，系统会在 RunLoop 的两个时机管理 AutoreleasePool：
- **kCFRunLoopEntry**：创建 Pool（`objc_autoreleasePoolPush`）
- **kCFRunLoopBeforeWaiting**：释放旧 Pool，创建新 Pool（`objc_autoreleasePoolPop` + `objc_autoreleasePoolPush`）
- **kCFRunLoopExit**：释放 Pool（`objc_autoreleasePoolPop`）

这就是为什么在 `for` 循环中创建大量临时对象时，需要手动加 `@autoreleasepool {}` 来及时释放内存。

### 6. 手势识别与触摸事件分发

触摸事件通过 Source0 进入 RunLoop：
1. `UIApplication` 收到触摸事件（Source1，基于 mach port）
2. 封装为 `UIEvent`，标记为待处理（Source0）
3. RunLoop 处理 Source0 时，将事件分发给 `UIWindow`
4. 通过 Hit-Test 找到目标 View
5. 调用 `touchesBegan:` / `touchesMoved:` 等方法

---

## 七、总结速查表

| 概念 | 关键点 |
|------|--------|
| **RunLoop 本质** | do-while 事件循环，保持线程存活，空闲时休眠省电 |
| **与线程关系** | 一一对应，主线程自动启动，子线程需手动启动 |
| **Mode** | Default / Tracking / Common / Initialization / System |
| **Source0** | 非 Port，手动触发（触摸、performSelector） |
| **Source1** | 基于 Port，系统内核唤醒（系统事件、mach msg） |
| **Timer** | 定时器，受 Mode 影响，需要加到 CommonMode 才能在滑动时触发 |
| **Observer** | 监听 RunLoop 状态变化（进入/退出/休眠/唤醒等） |
| **AutoreleasePool** | 在 Entry 创建，BeforeWaiting 重建，Exit 销毁 |
| **卡顿检测** | 监听 BeforeSources → BeforeWaiting 的时间差 |
