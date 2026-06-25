---
type: review
related_to: "[[内存管理]]", "[[Runtime]]", "[[RunLoop]]", "[[Block 闭包]]"
---

# AutoreleasePool 原理

## 一、什么是 AutoreleasePool

AutoreleasePool（自动释放池）是 Objective-C 内存管理的核心机制之一。它解决的核心问题是：**方法内部创建的对象，在方法返回时引用计数如何正确管理**。

```objc
- (NSString *)createGreeting {
    NSString *str = [[NSString alloc] initWithFormat:@"Hello, %@!", name];
    return str;
    // 问题：str 在方法返回后会怎样？
    // 如果方法内 release → 调用者拿到的是野指针
    // 如果方法内不 release → 调用者不知道何时释放
}
```

**解决方案**：将对象标记为"autorelease"，延迟到自动释放池销毁时统一 release。

## 二、底层数据结构

### 2.1 AutoreleasePoolPage

AutoreleasePool 底层是由 **双向链表** 连接的 `AutoreleasePoolPage` 节点：

```cpp
class AutoreleasePoolPage {
    magic_t const magic;          // 校验信息，检测内存损坏
    id *next;                     // 指向下一个可存放 autorelease 对象的位置
    pthread_t const thread;       // 所在线程（每个 Page 绑定一个线程）
    AutoreleasePoolPage * const parent;   // 父节点（双向链表）
    AutoreleasePoolPage *child;           // 子节点
    uint32_t const depth;         // 节点深度
    uint32_t hiwat;               // 水位标记（历史最大使用量）
    // 每个 Page 大小 4096 字节（一页虚拟内存）
};
```

**关键特性**：
- 每个 Page 大小 **4096 字节**（刚好一页虚拟内存）
- 使用**双向链表**连接，Page 满了就创建新 Page
- 每个线程有自己的 Page 链表
- `id *next` 指针从 Page 高地址向低地址增长（栈结构）

### 2.2 栈结构示意

```
AutoreleasePoolPage (4096 bytes)
┌─────────────────────────────┐ ← 高地址
│  magic_t (8 bytes)          │
│  id *next                   │ ← 指向下一个空位
│  pthread_t                  │
│  parent                     │
│  child                      │
│  depth                      │
│  hiwat                      │
│─────────────────────────────│
│  POOL_SENTINEL (nil)        │ ← @autoreleasepool {} 入口
│  autorelease obj 1          │
│  autorelease obj 2          │
│  autorelease obj 3          │
│  ...                        │
│                             │ ← next 指向这里
└─────────────────────────────┘ ← 低地址（向此处增长）
```

## 三、@autoreleasepool 编译后的本质

### 3.1 编译转换

`@autoreleasepool {}` 会被编译器转换为 `objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop`：

```objc
// 源代码
@autoreleasepool {
    NSString *str = [[NSString alloc] initWithFormat:@"Hello"];
    NSLog(@"%@", str);
}

// 编译后等价于：
void *pool = objc_autoreleasePoolPush();   // 压入哨兵对象
NSString *str = [[NSString alloc] initWithFormat:@"Hello"];
NSLog(@"%@", str);
objc_autoreleasePoolPop(pool);             // 弹出并 release 所有对象
```

### 3.2 Push 流程

```
objc_autoreleasePoolPush()
    ↓
autoreleaseFast(page, POOL_SENTINEL)
    ↓
Page 有空间？→ page->add(POOL_SENTINEL)  → 返回哨兵地址
    ↓ 无空间
autoreleaseFullPage(POOL_SENTINEL, page)
    ↓
创建新 Page → 新 Page->add(POOL_SENTINEL) → 返回哨兵地址
```

- **POOL_SENTINEL** 是 `nil`，它是一个标记，标记自动释放池的边界
- Push 的返回值是哨兵对象的地址，后续 Pop 时用来定位释放范围

### 3.3 Pop 流程

```
objc_autoreleasePoolPop(pool)
    ↓
找到 pool 对应的 Page
    ↓
从 next 指针开始，向前遍历，逐个 release 对象
    ↓
遇到 POOL_SENTINEL 停止
    ↓
重置 next 指针
```

### 3.4 autorelease 对象加入流程

```
[obj autorelease]
    ↓
autoreleaseFast(当前 Page, obj)
    ↓
Page 有空间？→ page->add(obj) → 返回 obj
    ↓ 无空间
autoreleaseFullPage(obj, page)
    ↓
找到有空间的 Page 或创建新 Page → page->add(obj) → 返回 obj
```

## 四、RunLoop 与 AutoreleasePool 的关系

### 4.1 RunLoop 回调机制

App 启动后，系统在主线程 RunLoop 中注册了两个回调：

```cpp
// RunLoop 进入前 Push
static void _wrapRunLoopWithAutoreleasePoolHandler(CFRunLoopActivity activity, void *unused) {
    if (activity == kCFRunLoopEntry) {
        // 即将进入 Loop → Push 一个 autoreleasepool
        objc_autoreleasePoolPush();
    }
}

// RunLoop 即将休眠 / 退出时处理
static void _wrapRunLoopWithAutoreleasePoolHandler(CFRunLoopActivity activity, void *unused) {
    if (activity == kCFRunLoopBeforeWaiting) {
        // 即将休眠 → Pop + Push（释放旧池，创建新池）
        objc_autoreleasePoolPop();
        objc_autoreleasePoolPush();
    }
    if (activity == kCFRunLoopExit) {
        // 即将退出 → Pop
        objc_autoreleasePoolPop();
    }
}
```

### 4.2 一次 RunLoop 迭代中的 autoreleasepool 生命周期

```
RunLoop 开始
    ↓
Push autoreleasepool（创建池）
    ↓
处理事件/Timer/Source/Block
    ↓
即将休眠 → Pop autoreleasepool（释放池中对象）
            → Push autoreleasepool（创建新池）
    ↓
被唤醒 → 处理新事件
    ↓
即将休眠 → Pop → Push ...
    ↓
RunLoop 退出 → Pop（最终释放）
```

**实际效果**：
- **一次 RunLoop 迭代** = 一个 autoreleasepool 的生命周期
- 迭代结束时释放所有 autorelease 对象，避免内存峰值
- 这就是为什么在主线程中，autorelease 对象通常在当前 RunLoop 迭代结束后释放

## 五、autorelease 对象的释放时机

### 5.1 不同场景的释放时机

| 场景 | 释放时机 |
|------|----------|
| **主线程普通代码** | 当前 RunLoop 迭代结束（即将休眠时） |
| **手动 @autoreleasepool {}** | 离开 `@autoreleasepool {}` 作用域时 |
| **子线程** | 子线程 RunLoop 结束或手动管理 |
| **GCD Block 中** | Block 执行完后（系统会管理） |

### 5.2 MRC vs ARC 下的区别

**MRC**：需要手动调用 `[obj autorelease]`

```objc
// MRC
- (NSString *)createString {
    NSString *str = [[[NSString alloc] initWithFormat:@"hello"] autorelease];
    return str; // 调用者不需要关心释放
}
```

**ARC**：编译器自动插入 autorelease

```objc
// ARC
- (NSString *)createString {
    NSString *str = [[NSString alloc] initWithFormat:@"hello"];
    return str; // 编译器自动插入 autorelease（在某些情况下）
}
```

ARC 下编译器的优化策略：
- 如果返回的对象有 `__strong` 修饰，且方法名以 `alloc`/`new`/`copy`/`mutableCopy` 开头 → 直接返回，不 autorelease
- 否则 → 调用 `objc_autoreleaseReturnValue` + `objc_retainAutoreleasedReturnValue` 配对优化

## 六、实际应用与优化

### 6.1 循环中创建大量临时对象

**问题**：循环中产生大量临时对象，内存峰值过高

```objc
// ❌ 内存峰值高
for (int i = 0; i < 1000000; i++) {
    NSString *str = [NSString stringWithFormat:@"item_%d", i];
    // str 会在当前 RunLoop 迭代结束后才释放
    // 100万个对象同时存活 → 内存爆掉
}
```

**解决方案**：手动加 autoreleasepool

```objc
// ✅ 每次循环释放临时对象
for (int i = 0; i < 1000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"item_%d", i];
        // 离开 @autoreleasepool {} 时立即释放
    }
}
```

### 6.2 子线程中的 autoreleasepool

子线程默认没有 RunLoop，需要手动管理：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    // 子线程中大量临时对象
    @autoreleasepool {
        for (int i = 0; i < 100000; i++) {
            @autoreleasepool {
                id obj = [self createObject];
                // 使用 obj
            }
        }
    }
});
```

### 6.3 常见的 autorelease 使用场景

```objc
// 1. 工厂方法返回对象（ARC 自动处理）
+ (instancetype)personWithName:(NSString *)name {
    return [[self alloc] initWithName:name];
    // ARC 下编译器自动处理，不需要手动 autorelease
}

// 2. 容器遍历中的临时对象
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    @autoreleasepool {
        // 大量临时对象创建
    }
}];

// 3. 图片处理
for (UIImage *image in images) {
    @autoreleasepool {
        UIImage *processed = [self processImage:image];
        // 处理完立即释放原图
    }
}
```

## 七、AutoreleasePool 的嵌套

```objc
@autoreleasepool {                    // Pool A
    NSString *objA = [NSString stringWithFormat:@"A"];
    @autoreleasepool {                // Pool B
        NSString *objB = [NSString stringWithFormat:@"B"];
        @autoreleasepool {            // Pool C
            NSString *objC = [NSString stringWithFormat:@"C"];
            // Pool C Pop 时释放 objC
        }
        // Pool B Pop 时释放 objB
    }
    // Pool A Pop 时释放 objA
}
```

**栈结构**：
```
[next] → objC → objB → POOL_SENTINEL → objA → POOL_SENTINEL
         ↑ C Pop     ↑ B Pop           ↑ A Pop
```

## 八、验证与调试

### 8.1 查看 autoreleasepool 行为

```objc
// 打印对象引用计数变化
NSObject *obj = [[NSObject alloc] init];
NSLog(@"retainCount after alloc: %lu", [obj retainCount]); // 1

@autoreleasepool {
    [obj autorelease];
    NSLog(@"retainCount in pool: %lu", [obj retainCount]); // 1（autorelease 不改变 retainCount）
}

NSLog(@"retainCount after pool: %lu", [obj retainCount]); // 可能已释放
```

### 8.2 使用 Instruments 检测

- **Allocations**：观察 autorelease 对象的生命周期
- **Leaks**：检测因 autoreleasepool 管理不当导致的泄漏
- **Memory Graph**：查看对象的实际释放时机

### 8.3 查看 RunLoop 的 autoreleasepool 行为

```objc
// 添加 RunLoop Observer 观察 autoreleasepool 的 Push/Pop
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(
    kCFAllocatorDefault,
    kCFRunLoopAllActivities,
    YES, 0,
    ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case kCFRunLoopEntry:
                NSLog(@"RunLoop 进入 → autoreleasepool Push");
                break;
            case kCFRunLoopBeforeWaiting:
                NSLog(@"RunLoop 即将休眠 → autoreleasepool Pop + Push");
                break;
            case kCFRunLoopExit:
                NSLog(@"RunLoop 退出 → autoreleasepool Pop");
                break;
        }
    });
CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
```

## 九、面试回答模板

> **面试官：讲一下 AutoreleasePool 的原理？**

我会从以下几个层次回答：

**1. 是什么**
AutoreleasePool 是 OC 内存管理的核心机制，解决方法内部创建的对象如何延迟释放的问题。对象调用 autorelease 后，会在自动释放池销毁时统一 release。

**2. 底层结构**
底层是双向链表连接的 AutoreleasePoolPage，每个 Page 大小 4096 字节，使用栈结构存储 autorelease 对象。@autoreleasepool {} 编译后变成 objc_autoreleasePoolPush 和 objc_autoreleasePoolPop 两个函数调用，Push 时插入一个 nil 哨兵对象，Pop 时释放哨兵之后的所有对象。

**3. 与 RunLoop 的关系**
主线程 RunLoop 注册了回调：进入时 Push，即将休眠时 Pop + Push，退出时 Pop。所以主线程的 autorelease 对象在每次 RunLoop 迭代结束时释放。

**4. 使用场景**
最常见的场景是循环中创建大量临时对象，需要手动加 @autoreleasepool {} 控制内存峰值。子线程也需要手动管理，因为子线程默认没有 RunLoop。

**5. ARC 下的变化**
ARC 下编译器会自动在合适的位置插入 autorelease，并且有 objc_autoreleaseReturnValue + objc_retainAutoreleasedReturnValue 的配对优化，减少不必要的 autorelease 操作。

---

**相关笔记**：[[内存管理]]、[[Runtime]]、[[RunLoop]]、[[Block 闭包]]
