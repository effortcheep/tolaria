---
type: review
---

# @property 本质

## 一句话回答

`@property = _ivar（实例变量） + getter + setter`。编译器根据属性声明自动生成带下划线前缀的实例变量和对应的存取方法，同时在 ARC 下自动管理内存语义（`strong`/`weak`/`copy`）。

---

## 1. @property 的发展三阶段

### 阶段一：手动声明（Xcode 4.4 之前）

```objc
// .h 中声明方法
@interface Person : NSObject
- (NSString *)name;
- (void)setName:(NSString *)name;
@end

// .m 中声明实例变量 + 实现方法
@implementation Person {
    NSString *_name;
}
- (NSString *)name {
    return _name;
}
- (void)setName:(NSString *)name {
    _name = name;
}
@end
```

### 阶段二：@property + @synthesize

```objc
// .h
@property (nonatomic, copy) NSString *name;

// .m — 必须手动 @synthesize
@synthesize name = _name; // 指定实例变量名
```

`@synthesize` 的作用：**自动生成 getter/setter 实现**，并创建对应的实例变量。

### 阶段三：@property only（Xcode 4.4+，现在）

```objc
// .h
@property (nonatomic, copy) NSString *name;

// .m — 什么都不用写，@synthesize 自动执行
// 等价于自动创建了：
//   1. 实例变量 _name
//   2. - (NSString *)name { return _name; }
//   3. - (void)setName:(NSString *)name { _name = [name copy]; }
```

> **面试要点**：从 Xcode 4.4 开始，`@synthesize` 默认自动执行，`@property` 一句话 = 声明 + 实现 + 实例变量。

---

## 2. @property 编译器本质展开

```objc
@property (nonatomic, copy) NSString *name;
```

编译器实际做的事情：

```objc
// 1. 生成实例变量
NSString *_name;

// 2. 生成 getter
- (NSString *)name {
    return _name;
}

// 3. 生成 setter（根据内存语义不同而不同）
- (void)setName:(NSString *)name {
    _name = [name copy];  // copy 语义
}
```

### 不同内存语义的 setter 实现

```objc
// strong 语义
- (void)setName:(NSString *)name {
    [name retain];
    [_name release];
    _name = name;
    // ARC 下编译器自动插入 retain/release
}

// copy 语义
- (void)setName:(NSString *)name {
    NSString *copy = [name copy];
    [_name release];
    _name = copy;
}

// weak 语义
- (void)setName:(NSString *)name {
    objc_storeWeak(&_name, name);  // 存入 SideTable
}

// assign 语义（基本数据类型）
- (void)setAge:(NSInteger)age {
    _age = age;  // 直接赋值，不涉及引用计数
}
```

---

## 3. 属性关键字详解

### 3.1 原子性

| 关键字 | 线程安全 | 实现方式 | 性能 |
|--------|----------|----------|------|
| `atomic`（默认） | getter/setter 原子性 | 自旋锁（spinlock） | 较慢 |
| `nonatomic` | 无锁保护 | 直接读写 | 快 |

```objc
// atomic 的本质（简化）
- (NSString *)name {
    @synchronized(self) {  // 实际用自旋锁
        return _name;
    }
}
```

> **注意**：`atomic` 只保证单次读写的原子性，不保证线程安全。`NSMutableArray *array` 即使是 atomic，多线程同时读写仍会 crash。

### 3.2 读写属性

| 关键字 | 生成的方法 |
|--------|-----------|
| `readwrite`（默认） | getter + setter |
| `readonly` | 仅 getter |

```objc
@property (nonatomic, readonly) NSString *name;
// 只生成 - (NSString *)name;
// 不生成 setter，外部无法赋值
```

### 3.3 内存管理语义

| 关键字 | 适用类型 | setter 行为 | 底层实现 |
|--------|---------|------------|---------|
| `strong` | OC 对象 | retain 新值，release 旧值 | `objc_storeStrong` |
| `weak` | OC 对象 | 不 retain，对象销毁时自动置 nil | `objc_storeWeak` |
| `copy` | OC 对象 | copy 新值，release 旧值 | `[obj copy]` |
| `assign` | 基本类型 | 直接赋值 | `_ivar = value` |
| `unsafe_unretained` | OC 对象 | 不 retain，不置 nil | 直接赋值 |

### copy 与 strong 的区别（面试高频）

```objc
@property (nonatomic, strong) NSString *strongStr;
@property (nonatomic, copy) NSString *copyStr;

NSMutableString *mutStr = [NSMutableString stringWithFormat:@"hello"];
self.strongStr = mutStr;
self.copyStr = mutStr;

[mutStr appendString:@" world"];

NSLog(@"strongStr = %@", self.strongStr);  // "hello world" ← 被修改！
NSLog(@"copyStr = %@", self.copyStr);      // "hello" ← 不受影响
```

> **规则**：`NSString` / `NSArray` / `NSDictionary` / `NSSet` 用 `copy`，防止传入可变子类导致数据被外部修改。

---

## 4. @synthesize 的高级用法

### 指定实例变量名

```objc
@synthesize name = _myName;  // 不用默认的 _name
// getter 方法中访问 _myName
```

### 在 .h 中声明 @property，在 .m 中指定 @synthesize

```objc
// .h
@property (nonatomic, copy) NSString *name;

// .m
@synthesize name = _name;  // 显式指定（通常不需要）
```

### 在分类（Category）中使用 @property

```objc
// Category 中 @property 不会自动生成实例变量和 setter
// 只会生成方法声明，需要借助关联对象实现
@interface Person (Extension)
@property (nonatomic, copy) NSString *nickname;
@end

@implementation Person (Extension)
- (NSString *)nickname {
    return objc_getAssociatedObject(self, @selector(nickname));
}
- (void)setNickname:(NSString *)nickname {
    objc_setAssociatedObject(self, @selector(nickname), nickname, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
@end
```

---

## 5. @property 与 Runtime

### 属性的元数据

```objc
// 获取类的所有属性
unsigned int count;
objc_property_t *properties = class_copyPropertyList([Person class], &count);

for (int i = 0; i < count; i++) {
    objc_property_t property = properties[i];
    const char *name = property_getName(property);
    const char *attrs = property_getAttributes(property);
    NSLog(@"%s -> %s", name, attrs);
    // 输出: name -> T@"NSString",C,N,V_name
    // T: type  C: copy  N: nonatomic  V: backing ivar
}
```

### property_getAttributes 编码

| 编码 | 含义 |
|------|------|
| `T` | 类型编码（如 `T@"NSString"`） |
| `R` | readonly |
| `C` | copy |
| `&` | strong (retain) |
| `W` | weak |
| `N` | nonatomic |
| `D` | dynamic（@dynamic） |
| `V_name` | 实例变量名为 `_name` |

### @dynamic

```objc
@dynamic name;
// 告诉编译器：getter/setter 由开发者手动实现或由 Runtime 动态添加
// 常用于 Core Data 的 NSManagedObject 子类
```

---

## 6. 常见陷阱

### 6.1 Block 属性用 copy

```objc
// ❌ 错误
@property (nonatomic, strong) void (^myBlock)(void);

// ✅ 正确
@property (nonatomic, copy) void (^myBlock)(void);
```

Block 默认在栈上，`copy` 后才会复制到堆上。`strong` 也能工作（编译器优化），但语义上 `copy` 更准确。

### 6.2 delegate 用 weak

```objc
// ❌ 错误 — 循环引用
@property (nonatomic, strong) id<MyDelegate> delegate;

// ✅ 正确
@property (nonatomic, weak) id<MyDelegate> delegate;
```

### 6.3 NSString 用 copy

```objc
// ❌ 危险 — 外部可变字符串修改会影响属性值
@property (nonatomic, strong) NSString *name;

// ✅ 安全
@property (nonatomic, copy) NSString *name;
```

### 6.4 @property 在 .h 和 .m 中的选择

```objc
// .h — 需要对外暴露的接口
@property (nonatomic, readonly) NSString *name;

// .m class extension — 内部可写
@interface Person ()
@property (nonatomic, readwrite, copy) NSString *name;
@end
```

---

## 7. 面试回答模板

> **Q：@property 的本质是什么？**
>
> `@property` 本质上是三样东西的组合：**实例变量 + getter + setter**。
>
> 从 Xcode 4.4 开始，`@property` 一行代码等价于：自动生成带下划线前缀的实例变量、getter 方法实现、setter 方法实现。在 ARC 下，编译器还会根据内存语义（`strong`/`weak`/`copy`）自动在 setter 中插入引用计数管理代码。
>
> 属性的关键字分为三类：**原子性**（atomic/nonatomic）、**读写权限**（readwrite/readonly）、**内存语义**（strong/weak/copy/assign）。
>
> 一些值得注意的细节：
> - `NSString` 和集合类型用 `copy`，防止外部可变子类修改数据
> - `Block` 属性用 `copy`，把栈上的 Block 拷贝到堆上
> - `delegate` 用 `weak`，避免循环引用
> - `atomic` 只保证单次读写的原子性，不等于线程安全
>
> 底层实现上，`@property` 通过 `property_getAttributes()` 可以读取到类型编码、内存语义等元数据，Runtime 的关联对象机制可以在 Category 中为属性添加存储能力。

---

## 相关笔记

- [[Runtime]] — 属性元数据、`class_copyPropertyList`、关联对象
- [[Category 与 Extension]] — Category 中 @property 的实现
- [[内存管理]] — strong/weak/copy 的引用计数机制
- [[Block 闭包]] — Block 为什么用 copy
- [[KVO 与 KVC]] — 属性的 KVC 赋值与 KVO 监听
