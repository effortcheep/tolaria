---
type: review
---
# Tagged Pointer

> **什么是 Tagged Pointer？它解决了什么问题？NSNumber、NSDate、小 NSString 是怎么用 Tagged Pointer 优化的？Tagged Pointer 有什么坑？**

## 一、什么是 Tagged Pointer？

**Tagged Pointer** 是 Apple 在 64 位系统上引入的一种内存优化技术，核心思想是：**把小对象的值直接编码进指针本身，而不是在堆上分配内存**。

### 传统对象的内存布局

```
栈上的 isa 指针  →  堆上的对象内存
┌──────────┐     ┌──────────────────┐
│ 0x123456 │────→│ isa              │
└──────────┘     │ retainCount      │
                 │ 实际值 (如 int=42)│
                 └──────────────────┘
```

- 每个对象至少需要 16 字节堆内存（isa + 值）
- 需要经过 `alloc → init → retain/release → dealloc` 完整生命周期
- 需要引用计数管理

### Tagged Pointer 的内存布局

```
指针本身直接存储值
┌──────────────────────┐
│ 0xB00000000000002A   │  ← 指针就是值，没有堆内存！
└──────────────────────┘
  │││               │
  标记位            实际值 (42)
```

- 最高位（bit 63）= 1 表示这是一个 Tagged Pointer
- 低 60 位直接存储数据
- **没有堆分配，没有引用计数，没有 dealloc**

## 二、它解决了什么问题？

### 1. 减少内存分配

```
// 传统方式：创建一个 NSNumber
NSNumber *num = [[NSNumber alloc] initWithInt:42];
// 堆上分配 16 字节 + isa + retainCount

// Tagged Pointer 方式
NSNumber *num = @42;
// 指针本身 = 值，0 字节堆分配
```

### 2. 消除引用计数开销

```
// 传统方式：需要 ARC 管理
id obj = [[NSObject alloc] init];
// retain +1, release -1, dealloc

// Tagged Pointer：不需要引用计数
NSNumber *num = @42;
// 直接复制指针即可，无需 retain/release
```

### 3. 提升性能

| 操作 | 传统对象 | Tagged Pointer |
|------|---------|----------------|
| 创建 | malloc + init | 直接编码指针 |
| 销毁 | release + dealloc | 什么都不做 |
| 复制 | retain +1 | 直接赋值 |
| 访问值 | 指针解引用 + 偏移 | 指针解码 |

## 三、NSNumber 的 Tagged Pointer 优化

### 原理

```
// NSNumber 内部实现（简化）
+ (NSNumber *)numberWithInt:(int)value {
    // 小整数直接用 Tagged Pointer
    if (value 在合理范围内) {
        // 指针 = 标记位 | value
        return (NSNumber *)(0xB000000000000000 | value);
    }
    // 大数值才真正分配堆内存
    return [[NSNumber alloc] initWithInt:value];
}
```

### 如何判断一个 NSNumber 是否是 Tagged Pointer？

```objc
// 方式一：通过 runtime 检查
NSNumber *num = @42;
NSLog(@"%p", num);  // 输出类似 0xb0000000000002a0
// 最高位是 b (1011)，说明是 Tagged Pointer

// 方式二：通过 isa 指针检查
// Tagged Pointer 没有真正的 isa，其 isa 指向特殊的类
Class cls = object_getClass(num);
NSLog(@"%@", cls);  // 输出 __NSCFNumber（Tagged Pointer 版本）
```

### 范围限制

```
// 小整数范围（通常）：-1 到 13（64位系统上可能更大）
NSNumber *a = @0;    // Tagged Pointer ✓
NSNumber *b = @42;   // Tagged Pointer ✓
NSNumber *c = @1000000; // 可能不是 Tagged Pointer ✗
```

## 四、NSDate 的 Tagged Pointer 优化

### 原理

```objc
// NSDate 内部（简化）
+ (NSDate *)dateWithTimeIntervalSinceReferenceDate:(NSTimeInterval)ti {
    // 小时间值直接用 Tagged Pointer
    if (ti 在合理范围内) {
        // 指针 = 0xA000000000000000 | 编码后的时间值
        return (NSDate *)(0xA000000000000000 | encodedTime);
    }
    // 超出范围才分配堆内存
    return [[NSDate alloc] initWithTimeIntervalSinceReferenceDate:ti];
}
```

### 验证

```objc
NSDate *date = [NSDate date];
NSLog(@"%p", date);  // 0xa 开头 → Tagged Pointer

// Tagged Pointer 的 NSDate 只能表示一定精度和范围的时间
// 超出范围会退化为普通对象
```

## 五、小 NSString 的 Tagged Pointer 优化

### 原理

```objc
// NSString 的 Tagged Pointer 编码（简化）
// 指针结构：
// [8 bits 标记][8 bits 长度][最多 6 bytes 字符数据]

// 例如 @"abc"：
// 0xE000000061626300
//   ││      ││││
//   标记    a b c
```

### 支持的字符范围

```
// ASCII 字符（0-127）可以使用 Tagged Pointer
NSString *s1 = @"abc";      // Tagged Pointer ✓
NSString *s2 = @"hello";    // Tagged Pointer ✓
NSString *s3 = @"1234567";  // 可能超出长度限制 ✗

// Unicode 字符通常不支持 Tagged Pointer
NSString *s4 = @"你好";      // 堆分配 ✗
```

### 关键限制

| 条件 | 是否 Tagged Pointer |
|------|-------------------|
| ASCII 字符 + 长度 ≤ 7 | ✅ 是 |
| ASCII 字符 + 长度 > 7 | ❌ 否 |
| 包含非 ASCII 字符 | ❌ 否 |
| 字面量 @"..." | 通常编译期优化 |

## 六、Tagged Pointer 的坑（⚠️ 面试重点）

### 坑 1：isa 指针问题

```objc
NSNumber *num = @42;

// ❌ 错误：直接访问 isa
Class cls = *((Class *)num);  // 崩溃！这不是真正的对象

// ✅ 正确：使用 runtime API
Class cls = object_getClass(num);
BOOL isTP = [num isMemberOfClass:cls];  // 行为可能不符合预期
```

### 坑 2：objc_msgSend 的处理

```objc
// Tagged Pointer 调用方法时，runtime 会特殊处理
NSNumber *num = @42;
NSString *str = [num stringValue];  
// runtime 检测到是 Tagged Pointer
// → 查找特殊的 Tagged Pointer 类
// → 通过指针解码值
// → 执行方法
```

### 坑 3：关联对象（Associated Object）崩溃

```objc
NSNumber *num = @42;

// ❌ 崩溃！Tagged Pointer 不是真正的对象
objc_setAssociatedObject(num, @"key", @"value", OBJC_ASSOCIATION_RETAIN);

// ✅ 先转成非 Tagged Pointer 对象
NSNumber *realNum = [[NSNumber alloc] initWithInt:42];
objc_setAssociatedObject(realNum, @"key", @"value", OBJC_ASSOCIATION_RETAIN);
```

### 坑 4：KVO 问题

```objc
// ❌ Tagged Pointer 对象不能添加 KVO
NSNumber *num = @42;
[num addObserver:self forKeyPath:@"value" options:0 context:nil]; 
// 崩溃或未定义行为

// ✅ KVO 需要真正的堆对象
```

### 坑 5：isKindOfClass 行为变化

```objc
NSNumber *num = @42;

// 某些情况下行为可能不一致
[num isKindOfClass:[NSNumber class]];  // ✅ 通常正常
[num isKindOfClass:[NSObject class]];  // ✅ 通常正常

// 但如果检查的是内部私有类...
Class cls = object_getClass(num);
// 可能是 __NSCFNumber，不是 NSNumber
// 导致某些类型检查失败
```

### 坑 6：内存地址比较

```objc
NSNumber *a = @42;
NSNumber *b = @42;

// ✅ Tagged Pointer 相同值 = 相同指针
a == b;  // YES

// 但传统对象不同
NSNumber *c = [[NSNumber alloc] initWithInt:42];
NSNumber *d = [[NSNumber alloc] initWithInt:42];
c == d;  // NO，不同堆对象

// ⚠️ 这可能导致依赖指针比较的代码出现 bug
```

### 坑 7：序列化/反序列化

```objc
// Tagged Pointer 在归档/解档时需要特殊处理
// 某些第三方库可能假设所有对象都在堆上

// 归档：通常工作正常
NSData *data = [NSKeyedArchiver archivedDataWithRootObject:@42];

// 但解档后的对象可能不再是 Tagged Pointer
NSNumber *num = [NSKeyedUnarchiver unarchiveObjectWithData:data];
// num 可能是堆对象，行为略有不同
```

## 七、runtime 如何识别 Tagged Pointer？

### 判断流程（简化）

```objc
// objc-class.mm 中的逻辑
static inline bool _objc_isTaggedPointer(const void *ptr) {
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) != 0;
}

// 64 位系统：最高位 = 1 → Tagged Pointer
#define _OBJC_TAG_MASK (1UL << 63)
```

### 消息发送流程

```
objc_msgSend(receiver, sel, ...)

1. receiver 是 nil？→ 返回 0
2. receiver 是 Tagged Pointer？
   ├── YES → 查找 Tagged Pointer 类表
   │         根据指针高 4 位确定类
   │         解码指针中的值
   │         调用方法实现
   └── NO  → 正常的 isa 查找流程
```

## 八、如何验证一个对象是否是 Tagged Pointer？

### 方法一：查看指针地址

```objc
id obj = ...;
NSLog(@"%p", obj);
// 0x8... → 普通堆对象（最高位 = 0）
// 0xb... → Tagged Pointer（最高位 = 1）
```

### 方法二：使用 runtime 函数

```objc
#import <objc/runtime.h>

// 在 iOS 11+ 可以使用
BOOL isTagged = _objc_isTaggedPointer(obj);
```

### 方法三：检查 isa

```objc
// 如果 object_getClass 返回的是特殊的 Tagged Pointer 类
// 如 __NSCFNumber、__NSCFString、__NSDate 等
// 且指针地址高位有特殊标记 → Tagged Pointer
```

## 九、面试回答模板

**面试官：什么是 Tagged Pointer？**

> Tagged Pointer 是 Apple 在 64 位系统上引入的内存优化技术。对于 NSNumber、NSDate、小 NSString 这类小对象，它把值直接编码进指针本身，而不是在堆上分配内存。
>
> 这样做的好处是：
> 1. 消除了 malloc/free 开销
> 2. 不需要引用计数管理
> 3. 方法调用更快（runtime 特殊处理）
>
> 但它也有一些坑：
> - 不能直接访问 isa 指针
> - 不能使用关联对象
> - 不能添加 KVO
> - 指针比较行为可能不符合预期
>
> 在实际开发中，我们通常不需要关心 Tagged Pointer，但在做底层调试、类型检查、或者写 Category 时需要注意这些边界情况。
