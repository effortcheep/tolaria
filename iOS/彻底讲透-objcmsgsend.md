---
type: review
related_to: "[[Runtime]]"
---
# 彻底讲透 objc_msgSend

## 一、什么是 objc_msgSend

`objc_msgSend` 是 OC 消息机制的核心函数，所有 OC 方法调用在编译后都会转换为对 `objc_msgSend` 的调用。

```objc
// OC 代码
[receiver message:arg];

// 编译后实际调用
objc_msgSend(receiver, @selector(message:), arg);
```

## 二、为什么需要 objc_msgSend

OC 是一门动态语言，方法调用在编译期无法确定具体执行哪个方法，需要在运行时通过 `objc_msgSend` 去查找并调用。

- **C 语言函数调用**：编译时确定地址，直接 `call` 指令跳转
- **OC 方法调用**：编译时只知道 selector，运行时才确定 IMP（函数指针）

## 三、objc_msgSend 的执行流程

### 1. 快速查找（Cache Lookup）

```
objc_msgSend(receiver, sel, ...)
    │
    ├── receiver == nil ? → return nil
    │
    └── 进入缓存查找
         ├── receiver->isa → 获取 class
         ├── class->cache → 查找 sel 对应的 IMP
         │    ├── cache 本质是哈希表 (bucket_t)
         │    ├── key = sel, value = IMP
         │    └── 命中 → 直接调用 IMP（最快路径）
         │
         └── 缓存未命中 → 进入慢速查找
```

### 2. 慢速查找（Method Lookup）

```
lookUpImpOrForward(cls, sel)
    │
    ├── 1. 检查类是否已初始化，未初始化则先 initialize
    │
    ├── 2. 当前类的方法列表（class_rw_t）
    │    ├── 二分查找 sorted method list
    │    └── 找到 → 缓存并返回 IMP
    │
    ├── 3. 沿继承链向上查找
    │    ├── superclass → superclass → ... → rootClass → nil
    │    └── 每一层都查 method list
    │
    ├── 4. 都没找到 → 进入动态方法解析
    │
    └── 5. 写入缓存，返回 IMP
```

### 3. 源码简化版

```cpp
IMP lookUpImpOrForward(Class cls, SEL sel, id inst,
                        bool initialize, bool cache) {
    Class curClass;
    IMP imp = nil;

    // 1. 先查缓存
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    // 2. 当前类和父类的方法列表
    curClass = cls;
    while (!done) {
        method_list_t *methods = curClass->baseMethods();
        imp = search_method_list(methods, sel);  // 二分查找
        if (imp) break;

        curClass = curClass->superclass;
        if (!curClass) {
            done = true;
            imp = nil;
        }
    }

    // 3. 找到后写入缓存
    cache_fill(cls, sel, imp);
    return imp;
}
```

## 四、方法缓存（Method Cache）

### 结构

```cpp
struct cache_t {
    struct bucket_t *_buckets;  // 哈希表
    mask_t _mask;               // 容量 - 1
    mask_t _occupied;           // 已占用数量
};

struct bucket_t {
    cache_key_t _key;   // SEL
    IMP _imp;            // 函数指针
};
```

### 哈希算法

```cpp
// 用 sel 的地址 & mask 得到索引
bucket_t *bucket = buckets + (sel & mask);
```

### 缓存膨胀（Cache Expand）

当 `_occupied > _mask * 3/4`（超过 75%），缓存会扩容：
- 新容量 = 旧容量 × 2（最小 4）
- 重新哈希所有 entry

### 缓存清理

- `cache_erase_nolock`：清空所有缓存
- 时机：method swizzling、category 加载等

## 五、objc_msgSend_stret

当返回值是大型结构体（超过寄存器能承载的大小），会使用 `objc_msgSend_stret`：

```objc
// 返回值是小结构体，用 objc_msgSend
CGPoint point = [view center];

// 返回值是大结构体，用 objc_msgSend_stret
CGRect rect = [view frame];
```

> **arm64 上**：不再区分，统一用 `objc_msgSend`，因为 arm64 寄存器足够大。

## 六、消息转发（找不到方法时）

当快速查找和慢速查找都找不到 IMP 时，进入消息转发流程：

```
lookUpImpOrForward 返回 nil
    │
    ├── 第一步：动态方法解析
    │    +resolveInstanceMethod: / +resolveClassMethod:
    │    └── 可以在这里动态添加方法实现
    │
    ├── 第二步：快速转发
    │    -forwardingTargetForSelector:
    │    └── 返回一个能处理该消息的对象
    │
    └── 第三步：完整转发
         -methodSignatureForSelector:
         -forwardInvocation:
         └── 最后的挽救机会
    │
    └── 都没实现 → doesNotRecognizeSelector: → crash
```

### 代码示例

```objc
// 1. 动态方法解析
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(test)) {
        class_addMethod(self, sel, (IMP)testMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

// 2. 快速转发
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) {
        return [[OtherClass alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}

// 3. 完整转发
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation invokeWithTarget:[[OtherClass alloc] init]];
}
```

## 七、性能优化

### 1. 缓存命中率

- OC 运行时会把调用过的方法缓存起来
- 同一个 selector 第二次调用就会命中缓存
- 缓存命中时的性能接近直接函数调用

### 2. 编译器优化

```objc
// 使用 __attribute__((objc_direct)) 标记直接调用
- (void)directMethod __attribute__((objc_direct)) {
    // 这个方法不会经过 objc_msgSend
}
```

### 3. Swift 的优化

Swift 方法默认是静态分派（不经过 `objc_msgSend`），除非：
- 标记了 `@objc`
- 继承自 NSObject
- 使用了协议（需要动态分派）

## 八、常见面试问题

### Q1: objc_msgSend 为什么用汇编实现？

因为需要：
- 最大化性能（减少函数调用开销）
- 保存所有寄存器状态（参数传递）
- 处理不同平台的 ABI 差异
- 支持尾调用优化（tail call）

### Q2: 缓存查找用什么数据结构？

哈希表（bucket_t 数组），用 `sel & mask` 作为 key。

### Q3: 方法查找的顺序？

1. 自己的方法列表
2. 父类的方法列表
3. 逐级向上直到 NSObject
4. 动态方法解析
5. 消息转发

### Q4: objc_msgSend 和普通函数调用的性能差距？

- 缓存命中时：接近 1:1（约 1.2 倍）
- 缓存未命中时：差距明显
- 整体来看，因为缓存机制，实际差距很小

## 九、总结

| 阶段 | 动作 | 结果 |
|------|------|------|
| 缓存查找 | 哈希表 O(1) | 命中 → 直接调用 |
| 慢速查找 | 遍历方法列表 | 找到 → 缓存并调用 |
| 动态解析 | resolveInstanceMethod | 添加方法 → 重走查找 |
| 快速转发 | forwardingTargetForSelector | 转发给其他对象 |
| 完整转发 | forwardInvocation: | 最后挽救 |
| 崩溃 | doesNotRecognizeSelector | crash |

**核心思想**：通过缓存机制，把消息传递的性能优化到接近直接调用的水平，同时保留动态语言的灵活性。
