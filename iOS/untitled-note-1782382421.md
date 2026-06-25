---
type: review
---
# Block 闭包

> **Block 是什么？Block 捕获外部变量的机制是什么？**`__block` **的作用是什么？Block 的内存位置有哪几种？**

## 一、Block 是什么

Block 是 Objective-C 中的**匿名函数**（闭包），本质上是一个**封装了函数指针和上下文环境的 Objective-C 对象**。

```objc
// 最简单的 Block
void (^myBlock)(void) = ^{
    NSLog(@"Hello Block");
};
myBlock();  // 调用

// 带参数和返回值
int (^add)(int, int) = ^int(int a, int b) {
    return a + b;
};
int result = add(1, 2);  // 3
```

### 1.1 Block 的本质结构

Block 编译后会被转换为一个 C 结构体：

```c
struct __block_impl {
    void *isa;              // 指向 Block 的类（_NSConcreteStackBlock 等）
    int Flags;
    int Reserved;
    void *invoke;           // 函数指针，指向 Block 的执行代码
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    // 捕获的外部变量作为结构体成员
    int capturedVar;
};
```

关键点：
- `isa` 指针说明 **Block 本质上是一个 ObjC 对象**，可以发送 `copy`、`release` 等消息
- `invoke` 是函数指针，指向 Block 花括号内的代码
- 捕获的外部变量作为结构体的**成员变量**存入 Block 对象

### 1.2 Block 的变量捕获机制

| 变量类型 | 捕获方式 | 能否修改 |
|----------|----------|----------|
| 局部变量（auto） | **值捕获**（拷贝一份到 Block 结构体） | ❌ 只读 |
| 局部静态变量 | **指针捕获**（捕获地址） | ✅ 可修改 |
| 全局变量 | **不捕获**（直接访问） | ✅ 可修改 |
| 全局静态变量 | **不捕获**（直接访问） | ✅ 可修改 |

```objc
int localVar = 10;
static int staticLocalVar = 20;
int globalVar = 30;

void (^block)(void) = ^{
    // localVar 是值捕获，Block 创建时拷贝了一份，之后外部修改不影响 Block 内的值
    NSLog(@"localVar = %d", localVar);       // 10（拷贝）
    NSLog(@"staticLocalVar = %d", staticLocalVar); // 指针捕获
    NSLog(@"globalVar = %d", globalVar);     // 直接访问
};

localVar = 100;
staticLocalVar = 200;
globalVar = 300;

block();
// 输出：localVar = 10, staticLocalVar = 200, globalVar = 300
```

**为什么局部变量是值捕获？**
- 局部变量（auto 变量）存储在**栈**上，函数返回后就会被销毁
- Block 可能在函数返回后才执行（比如异步回调）
- 如果只捕获指针，函数返回后指针就变成悬垂指针（dangling pointer）
- 所以必须在 Block 创建时**拷贝一份值**到堆上

**为什么全局变量不捕获？**
- 全局变量的生命周期是整个程序运行期间，任何时候都可访问
- 不存在生命周期问题，不需要捕获

---

## 二、`__block` 修饰符

### 2.1 `__block` 的作用

`__block` 解决了**在 Block 内部修改被捕获的局部变量**的问题。

```objc
// ❌ 编译错误！
int count = 0;
void (^block)(void) = ^{
    count = 10;  // 编译报错：Variable is not assignable
};

// ✅ 使用 __block
__block int count = 0;
void (^block)(void) = ^{
    count = 10;  // OK
};
block();
NSLog(@"count = %d", count);  // 10
```

### 2.2 `__block` 的底层原理

`__block` 会将变量包装成一个**对象**（`__Block_byref_xxx` 结构体），Block 捕获的是这个对象的**指针**：

```c
// 原始代码
__block int count = 0;

// 编译后（简化）
struct __Block_byref_count_0 {
    void *__isa;
    struct __Block_byref_count_0 *__forwarding;  // 指向自己（堆上）
    int __flags;
    int __size;
    int count;  // 实际的值存在这里
};

// Block 结构体里存的是 __Block_byref_count_0 的指针
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0 *Desc;
    struct __Block_byref_count_0 *count;  // 指针！不是值拷贝
};
```

关键点：
- `__block` 变量被包装成**堆对象**，Block 内外通过 `__forwarding` 指针访问同一个堆对象
- 所以 Block 内部修改 `count`，外部也能看到变化
- `__forwarding` 的作用：栈上的 `__forwarding` 指向堆上的对象，保证无论从栈还是堆访问，都访问的是堆上的同一份数据

### 2.3 `__block` 变量的 copy 行为

| Block 类型 | `__block` 变量位置 |
|------------|-------------------|
| 栈 Block | `__block` 变量在栈上 |
| 堆 Block（copy 后） | `__block` 变量被 copy 到堆上 |

多次 copy 同一个 Block 时，`__block` 变量不会重复 copy，而是引用计数 +1。所有 Block 共享同一个 `__block` 变量。

---

## 三、Block 的内存位置（三种类型）

### 3.1 三种 Block 类型

| 类型 | 存储位置 | 特征 | 生命周期 |
|------|----------|------|----------|
| **`_NSConcreteGlobalBlock`** | 全局区（Global/Data） | 不捕获任何自动变量 | 程序运行期间 |
| **`_NSConcreteStackBlock`** | 栈（Stack） | 捕获了自动变量，但未 copy | 随栈帧销毁 |
| **`_NSConcreteMallocBlock`** | 堆（Heap） | 栈 Block 被 copy 后 | 引用计数管理 |

### 3.2 如何判断 Block 类型

```objc
// 1. 全局 Block（Global Block）
// 不捕获任何自动变量
void (^globalBlock)(void) = ^{
    NSLog(@"没有捕获自动变量");
};
// 类型：_NSConcreteGlobalBlock
// 存在全局区，程序运行期间一直存在

// 2. 栈 Block（Stack Block）
// 捕获了自动变量，但未被 copy
int x = 10;
void (^stackBlock)(void) = ^{
    NSLog(@"x = %d", x);
};
// MRC 下类型：_NSConcreteStackBlock
// ARC 下编译器会自动 copy，打印可能是 _NSConcreteMallocBlock

// 3. 堆 Block（Malloc Block）
// 栈 Block 被 copy 后
void (^mallocBlock)(void) = [stackBlock copy];
// 类型：_NSConcreteMallocBlock
// 由引用计数管理，最安全
```

### 3.3 ARC 下的自动 copy

**ARC 下，编译器会自动将 Block 从栈 copy 到堆的情况：**

1. Block 作为函数**返回值**
2. 赋值给 `__strong` 修饰的变量（即强引用）
3. 作为 Cocoa API 中 `usingBlock:` 参数的方法参数
4. 作为 GCD API 的参数（如 `dispatch_async`）

```objc
// ARC 下自动 copy
- (void(^)(void))getBlock {
    int x = 10;
    // 返回值 Block，编译器自动 copy 到堆
    return ^{
        NSLog(@"x = %d", x);
    };
}

// 赋值给 __strong 变量，自动 copy
int y = 20;
void (^strongBlock)(void) = ^{
    NSLog(@"y = %d", y);
};  // ARC 下自动 copy 到堆
```

### 3.4 MRC 下必须手动 copy

```objc
// MRC 下，栈 Block 必须手动 copy 才能安全使用
int x = 10;
void (^stackBlock)(void) = ^{
    NSLog(@"x = %d", x);
};
// stackBlock 存在栈上，函数返回后就失效

void (^heapBlock)(void) = [stackBlock copy];  // copy 到堆
// heapBlock 在堆上，由引用计数管理，安全使用

[heapBlock release];  // 用完释放
```

---

## 四、Block 的 copy 与变量捕获的关系

### 4.1 对象类型的捕获

```objc
// 对象类型变量，默认也是值捕获（拷贝指针本身）
// 但 ARC 下会自动对捕获的对象加 __strong
NSObject *obj = [[NSObject alloc] init];

void (^block)(void) = ^{
    NSLog(@"%@", obj);  // 捕获的是 obj 指针的拷贝
    // ARC 下默认对 obj 有 __strong 引用
};

// obj 不会被释放，因为 Block 持有强引用
```

### 4.2 循环引用问题

```objc
// ❌ 循环引用
@interface ViewController ()
@property (nonatomic, copy) void (^myBlock)(void);
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    self.myBlock = ^{
        NSLog(@"%@", self);  // Block 捕获 self，self 持有 Block → 循环引用
    };
}
@end

// ✅ 方案一：使用 __weak 打破循环
__weak typeof(self) weakSelf = self;
self.myBlock = ^{
    NSLog(@"%@", weakSelf);  // 弱引用，不持有 self
};

// ✅ 方案二：使用 __weak + __strong 防止中途释放
__weak typeof(self) weakSelf = self;
self.myBlock = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        NSLog(@"%@", strongSelf);  // 执行期间 self 不会被释放
    }
};
```

### 4.3 `__block` 能否解决循环引用？

```objc
// ❌ __block 不能解决循环引用！
__block ViewController *blockSelf = self;
self.myBlock = ^{
    NSLog(@"%@", blockSelf);  // blockSelf 仍然强引用 self
};
// 只是多了一层间接引用，并没有打破循环

// ✅ __block + 手动置 nil（不推荐，容易忘记）
__block ViewController *blockSelf = self;
self.myBlock = ^{
    NSLog(@"%@", blockSelf);
    blockSelf = nil;  // 执行后打破循环，但只执行一次有效
};
self.myBlock();
```

---

## 五、Block 作为属性的正确写法

```objc
// ✅ 使用 copy 修饰
@property (nonatomic, copy) void (^myBlock)(void);

// 为什么用 copy？
// Block 默认在栈上，copy 后会到堆上，由引用计数管理
// 如果用 strong，ARC 下效果一样（编译器会自动 copy）
// 但用 copy 语义更明确，且与 MRC 兼容
```

---

## 六、Block 的实际应用场景

| 场景 | 示例 |
|------|------|
| **回调** | 网络请求完成、动画完成 |
| **GCD** | `dispatch_async(queue, ^{...})` |
| **排序** | `sortedArrayUsingComparator:` |
| **动画** | `[UIView animateWithDuration:animations:]` |
| **通知** | `addObserverForName:usingBlock:` |
| **Swift 闭包** | `@escaping` / `@Sendable` / `@MainActor` |

---

## 七、Swift 闭包与 OC Block 的对比

| 特性 | OC Block | Swift 闭包 |
|------|----------|-----------|
| 语法 | `^{...}` | `{...}` |
| 捕获变量 | 值捕获（默认），`__block` 可修改 | 默认可修改（闭包是引用类型） |
| 逃逸 | 默认非逃逸 | 默认非逃逸，`@escaping` 显式标注 |
| 循环引用 | `__weak typeof(self)` | `[weak self]` / `[unowned self]` |
| 尾随闭包 | 不支持 | 语法糖 `func { ... }` |
| 值捕获 | 变量值拷贝到闭包结构体 | 值类型捕获副本，引用类型捕获引用 |

```swift
// Swift 闭包捕获变量 — 默认可修改
var count = 0
let closure = {
    count += 1  // ✅ Swift 闭包默认允许修改捕获的变量
    print(count)
}
closure()  // 1
print(count)  // 1（外部也变了，因为捕获的是引用）

// Swift 值类型捕获
var array = [1, 2, 3]
let block = {
    array.append(4)  // ✅ 捕获的是 array 的引用（inout 效果）
}
block()
print(array)  // [1, 2, 3, 4]
```

---

## 八、总结速查表

| 概念 | 要点 |
|------|------|
| **Block 本质** | 匿名函数 + 捕获的上下文，编译后是 C 结构体，isa 指针说明它是 ObjC 对象 |
| **变量捕获** | 局部变量：值捕获；静态变量：指针捕获；全局变量：不捕获 |
| **`__block`** | 将变量包装为堆对象，Block 内外通过 `__forwarding` 指针访问同一份数据 |
| **三种内存位置** | 全局区（不捕获自动变量）、栈（捕获但未 copy）、堆（copy 后） |
| **ARC 自动 copy** | 返回值、`__strong` 变量、GCD/Cocoa API 参数 |
| **循环引用** | Block 捕获 self + self 持有 Block → 用 `__weak` / `[weak self]` 打破 |
| **属性用 copy** | 确保 Block 从栈 copy 到堆，由引用计数管理 |
| **Swift 闭包** | 默认可修改捕获变量、默认非逃逸、`[weak self]` 打破循环引用 |
