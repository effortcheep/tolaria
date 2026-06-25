---
type: review
---
# Runtime

> **iOS 的 Runtime 是什么？一条消息发送** `objc_msgSend` **的完整流程是怎样的？如果方法没找到会怎样？什么是 Method Swizzling，有什么应用场景？**

***

## 一、Runtime 是什么

Objective-C 是一门**动态语言**，它的动态性由 Runtime（运行时）库提供。Runtime 是一个用 **C + 汇编** 编写的共享库（`libobjc`），在程序运行时执行以下核心工作：

- **消息传递（Message Passing）**：ObjC 的方法调用本质是向对象发送消息
- **动态方法解析**：运行时决定方法的实现地址
- **动态类型识别**：`isKindOfClass:`、`respondsToSelector:` 等
- **Method Swizzling**：运行时替换方法实现
- **Associated Objects**：给分类添加属性
- **类的动态创建**：`objc_allocateClassPair` 运行时创建新类

> 一句话总结：**编译器只负责把** `[obj method]` **转换成** `objc_msgSend` **调用，具体调用哪个方法、能不能调用，全由 Runtime 在运行时决定。**

***

## 二、核心数据结构

```objc
// 1. objc_class（类）
struct objc_class : objc_object {
    Class isa;                // 元类的 isa
    Class superclass;         // 父类
    cache_t cache;            // 方法缓存（哈希表）
    class_data_bits_t bits;   // 存储 class_rw_t
};

// 2. class_rw_t（读写，运行时使用）
struct class_rw_t {
    const class_ro_t *ro;     // 只读数据（编译时确定）
    method_array_t methods;   // 方法列表（含分类）
    property_array_t properties;
    protocol_array_t protocols;
};

// 3. class_ro_t（只读，编译时确定）
struct class_ro_t {
    const char *name;
    method_list_t *baseMethodList;  // 本类的方法
    property_list_t *baseProperties;
    protocol_list_t *baseProtocols;
};

// 4. method_t（方法）
struct method_t {
    SEL name;        // 方法选择器（字符串）
    const char *types; // 参数和返回值编码
    IMP imp;         // 方法实现的函数指针
};
```

**关键关系**：

- 每个类有一个 `class_rw_t`，它持有本类 + 所有分类的方法列表
- `cache_t` 是一个哈希表，用来加速方法查找
- `SEL` 是方法名字符串，`IMP` 是函数指针，方法缓存就是 `SEL → IMP` 的映射

***

## 三、`objc_msgSend` 的完整流程

当你写下 `[person sayHello]`，编译器生成：

```asm
objc_msgSend(person, @selector(sayHello))
```

### 3.1 汇编入口（快速路径）

`objc_msgSend` 是用**汇编**写的，流程如下：

```text
1. 检查 receiver 是否为 nil
   └─ 如果是 nil → 直接返回 0（这就是为什么发消息给 nil 不崩溃）

2. 从 receiver 获取 isa → 得到 Class

3. 在 Class 的 cache 中查找 SEL
   └─ cache_t 是哈希表：sel >> 哈希位移 → slot
   └─ 如果命中 → 直接跳转到 IMP（快速路径，O(1)）

4. cache 未命中 → 调用 _objc_msgSend_uncached（慢速路径）
```

### 3.2 慢速查找（C++ 实现）

```text
5. lookUpImpOrForward(receiver, sel, cls)
   │
   ├─ 1. 加锁（防止并发问题）
   │
   ├─ 2. 检查类是否已实现（如果 cls 没有实现，先 realizeClass）
   │
   ├─ 3. 当前类的方法列表中查找
   │     └─ 二分查找 class_rw_t 的 methods 数组
   │     └─ 找到 → 缓存到 cache_t → 返回 IMP
   │
   ├─ 4. 沿继承链向上查找
   │     └─ superclass → superclass → superclass → ...
   │     └─ 每一层都查 methods 列表
   │     └─ 找到 → 缓存到当前类的 cache → 返回 IMP
   │
   ├─ 5. 到达根类（NSObject）仍未找到
   │     └─ 进入「方法解析」阶段
   │
   └─ 6. 解锁
```

**简化流程图**：

```javascript
objc_msgSend(receiver, sel)
  │
  ├─ receiver == nil? → return 0
  │
  ├─ cache_hit(sel)? → jump IMP (快速路径)
  │
  └─ cache_miss
       │
       ├─ 当前类 method_list 中查找 → 找到? → 缓存 → return IMP
       │
       ├─ superclass 的 method_list → 找到? → 缓存 → return IMP
       │
       ├─ ... 沿继承链向上 ...
       │
       └─ 全部未找到 → 动态方法解析
```

***

## 四、方法未找到：三个「救援」阶段

当 `objc_msgSend` 遍历完整条继承链仍未找到方法实现时，会依次进入三个阶段：

### 4.1 阶段一：动态方法解析（+resolveInstanceMethod:）

```objc
// 运行时给你一次机会，动态添加方法实现
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(dynamicMethod)) {
        // 动态添加方法
        class_addMethod(self, sel, (IMP)dynamicMethodIMP, "v@:");
        return YES;  // 返回 YES → 重新查找该方法
    }
    return [super resolveInstanceMethod:sel];
}

// C 函数作为实现
void dynamicMethod(id self, SEL _cmd) {
    NSLog(@"动态添加的方法");
}
```

> 典型场景：**@dynamic** 属性、KVO 的 setter 动态生成。

### 4.2 阶段二：消息转发（-forwardingTargetForSelector:）

```objc
// 把消息转发给另一个对象
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(sayHello)) {
        return self.helper;  // 转发给 helper 对象
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

> 快速转发：直接指定另一个对象来处理，开销小。

### 4.3 阶段三：完整消息转发（-forwardInvocation:）

```objc
// 1. 生成方法签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(sayHello)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

// 2. 转发调用
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    // 可以修改目标、修改参数、修改返回值
    [anInvocation invokeWithTarget:self.helper];
}
```

### 4.4 最终失败

如果三个阶段都无法处理，Runtime 调用：

```objc
- (void)doesNotRecognizeSelector:(SEL)aSelector {
    // 抛出异常：unrecognized selector sent to instance
}
```

### 完整决策链：

```text
方法未找到
  │
  ├─ +resolveInstanceMethod: (动态方法解析)
  │     └─ 返回 YES → 重新走消息查找
  │     └─ 返回 NO ↓
  │
  ├─ -forwardingTargetForSelector: (快速转发)
  │     └─ 返回非 nil → objc_msgSend(返回值, sel)
  │     └─ 返回 nil ↓
  │
  ├─ -methodSignatureForSelector: + -forwardInvocation: (完整转发)
  │     └─ 有签名 → forwardInvocation: 处理
  │     └─ 无签名 ↓
  │
  └─ -doesNotRecognizeSelector: → crash
```

***

## 五、Method Swizzling

### 5.1 是什么

Method Swizzling 是在**运行时**交换两个方法实现的技术。核心 API：

```objc
// 获取方法（实例方法用 class，类方法用 meta class）
Method originalMethod = class_getInstanceMethod(cls, @selector(original));
Method swizzledMethod = class_getInstanceMethod(cls, @selector(swizzled));

// 交换实现
method_exchangeImplementations(originalMethod, swizzledMethod);
```

### 5.2 正确的 Swizzling 写法

```objc
@implementation UIViewController (Swizzle)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class cls = [self class];
        
        SEL originalSel = @selector(viewWillAppear:);
        SEL swizzledSel = @selector(xxx_viewWillAppear:);
        
        Method originalMethod = class_getInstanceMethod(cls, originalSel);
        Method swizzledMethod = class_getInstanceMethod(cls, swizzledSel);
        
        // 1. 先尝试添加方法（防止子类未实现时 swizzle 失败）
        BOOL didAddMethod = class_addMethod(cls,
                                             originalSel,
                                             method_getImplementation(swizzledMethod),
                                             method_getTypeEncoding(swizzledMethod));
        if (didAddMethod) {
            // 2. 替换成功，说明类本身没有实现该方法（从父类继承的）
            class_replaceMethod(cls,
                                swizzledSel,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            // 3. 类本身有实现，直接交换
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

// 注意：交换后方法名互换，这里实际调用的是原始的 viewWillAppear:
- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];  // 这里调用的是原始实现
    NSLog(@"viewWillAppear: 被追踪了");
}

@end
```

### 5.3 为什么要用 `+load` + `dispatch_once`

- `+load`：类被加载时调用（main 之前），每个类/分类的 `+load` 都会调用且只调用一次
- `dispatch_once`：防止子类 `+load` 重复交换（子类未 override 会调用父类的 `+load`）

### 5.4 应用场景

| 场景       | 示例                                      |
| -------- | --------------------------------------- |
| **全局埋点** | 交换 `viewWillAppear:` 追踪页面访问             |
| **防崩溃**  | 交换 `arrayWithObject:` 防止插入 nil crash    |
| **图片加载** | 交换 `imageNamed:` 加入缓存策略或日志              |
| **网络请求** | 交换 `NSURLSession` 方法添加统一 header         |
| **字体适配** | 交换 `systemFontOfSize:` 适配动态字体           |
| **导航栏**  | 交换 `pushViewController:animated:` 管理导航栈 |

### 5.5 ⚠️ 注意事项

1. **只交换自己的方法**：不要 swizzle 系统方法的实现本身，应该在自定义方法中调用原始实现
2. **用** `+load` **而非** `+initialize`：`+load` 保证只执行一次，`+initialize` 可能被子类触发
3. **注意继承问题**：子类如果未重写被 swizzle 的方法，用 `class_addMethod` 先判断
4. **命名避免冲突**：自定义方法加前缀，如 `xxx_viewWillAppear:`
5. **交换方法名后调用要小心**：交换后 `xxx_viewWillAppear:` 实际指向原始实现

***

## 六、其他 Runtime 核心 API

### 6.1 方法相关

```objc
// 获取方法列表
Method *methods = class_copyMethodList(cls, &count);

// 获取方法实现
IMP imp = method_getImplementation(method);

// 获取方法类型编码
const char *type = method_getTypeEncoding(method);

// 替换方法实现
IMP old = class_replaceMethod(cls, sel, newIMP, types);
```

### 6.2 属性相关

```objc
// 获取属性列表
objc_property_t *properties = class_copyPropertyList(cls, &count);

// 获取属性名
const char *name = property_getName(property);

// 获取属性特性（@property 的 attributes）
const char *attrs = property_getAttributes(property);
// 如 T@"NSString",&,N,V_name
```

### 6.3 关联对象（给分类加属性）

```objc
static const char kAssociatedKey;

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, &kAssociatedKey, name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, &kAssociatedKey);
}
```

### 6.4 动态创建类

```objc
// 创建新类（继承自 NSObject）
Class newClass = objc_allocateClassPair([NSObject class], "MyDynamicClass", 0);

// 添加实例变量
class_addIvar(newClass, "_name", sizeof(id), log2(sizeof(id)), "@");

// 添加方法
class_addMethod(newClass, @selector(name), (IMP)nameIMP, "@@:");

// 注册类
objc_registerClassPair(newClass);

// 使用
id obj = [[newClass alloc] init];
```

***

## 七、isa、Class 与元类（Meta-Class）

```text
实例对象 (instance)
  │ isa
  ▼
类对象 (class)         ← 存放：实例方法（-开头）、实例属性、协议
  │ isa
  ▼
元类 (meta-class)      ←  存放：类方法（+开头）
  │ isa
  ▼
根元类 (root meta-class)  ← 元类对象的类是根元类
  │ isa → 自身（形成闭环）
```

- **实例的 isa** → 类对象
- **类对象的 isa** → 元类
- **元类的 isa** → 根元类（NSObject 的元类）
- **根元类的 isa** → 自身
- **类方法查找**：通过 isa 找到元类，在元类的方法列表中查找

***

## 七补：Swift 的派发机制

Swift 是一门**静态优先**的语言，但它同时支持三种派发方式。理解这一点是理解 Swift 与 ObjC Runtime 交互的关键。

### 7b.1 三种派发方式

| 派发方式                       | 决定时机         | 速度            | 适用场景                                 |
| -------------------------- | ------------ | ------------- | ------------------------------------ |
| **静态派发（Static Dispatch）**  | 编译时          | ⚡ 最快，可内联      | `struct`、`enum`、`final`、`private` 方法 |
| **函数表派发（Table Dispatch）**  | 编译时确定表，运行时查表 | 🔵 较快         | `class` 的普通方法、`protocol` 方法          |
| **消息派发（Message Dispatch）** | 运行时          | 🐢 最慢，但支持动态特性 | `@objc`、`dynamic` 修饰的方法              |

### 7b.2 静态派发（Static Dispatch）

```swift
struct Point {
    var x: Double
    var y: Double
    
    // 静态派发：编译器直接内联调用，等同于 C 函数调用
    func distance(to other: Point) -> Double {
        let dx = x - other.x
        let dy = y - other.y
        return (dx * dx + dy * dy).squareRoot()
    }
}

// final class 也走静态派发
final class Calculator {
    func add(_ a: Int, _ b: Int) -> Int { a + b }  // 静态派发
}
```

**为什么快？**

- 编译器在编译期就知道调用哪个函数
- 可以直接**内联（inline）**，消除函数调用开销
- 不需要查表、不需要 `objc_msgSend` 的哈希查找

### 7b.3 函数表派发（Table Dispatch）

类似 C++ 的虚函数表（vtable），Swift class 的方法存放在 **vtable** 中：

```swift
class Animal {
    func speak() { print("...") }       // vtable slot 0
    func move()  { print("walking") }   // vtable slot 1
}

class Dog: Animal {
    override func speak() { print("Woof!") }  // 覆盖 slot 0
}

let dog: Animal = Dog()
dog.speak()  // 编译时确定 vtable，运行时查表 → "Woof!"
```

```typescript
Animal vtable:          Dog vtable:
┌─────────────┐        ┌─────────────┐
│ 0: speak()  │        │ 0: speak()  │  ← 被覆盖
│ 1: move()   │        │ 1: move()   │  ← 继承
└─────────────┘        └─────────────┘
```

**查找过程**：对象 → isa → 类 → vtable[index] → 直接调用，比 `objc_msgSend` 的 cache 查找快得多。

### 7b.4 消息派发（Message Dispatch）

只有显式标记 `@objc` 或 `dynamic` 的方法才会走 ObjC Runtime：

```swift
class NSObjectSubclass: NSObject {
    @objc func objcMethod() { }        // 走 objc_msgSend
    dynamic func dynamicMethod() { }   // 走 objc_msgSend
    func swiftMethod() { }              // 走 vtable（不是 objc_msgSend）
}
```

**为什么还要用消息派发？**

- 需要 Method Swizzling（只能作用于消息派发的方法）
- 需要响应 `performSelector:` 等 ObjC 动态特性
- KVO 只能观察 `@objc dynamic` 属性

### 7b.5 Swift 的派发规则总结

```typescript
方法调用
  │
  ├─ struct / enum 方法？ ──────────────→ 静态派发（可内联）
  │
  ├─ final class / final 方法？ ─────────→ 静态派发（可内联）
  │
  ├─ private / fileprivate 方法？ ────────→ 静态派发（无法被子类覆盖）
  │
  ├─ @objc / dynamic 修饰？ ────────────→ 消息派发（objc_msgSend）
  │
  ├─ class / protocol 的普通方法？ ──────→ 函数表派发（vtable）
  │
  └─ @objc 协议的可选方法？ ────────────→ 消息派发（需要运行时查表）
```

### 7b.6 编译器优化：WMO 和 Speculative Devirtualization

Swift 编译器在**全模块优化（WMO）** 和 **-O** 优化下，会尝试把 vtable 调用**去虚拟化（devirtualize）** 为静态调用：

```swift
// 编译器如果能推断出具体类型，会将 vtable 调用优化为静态调用
func makeNoise(_ animal: Animal) {
    animal.speak()  // 本来是 vtable 查表
}

// 如果编译器能证明这里只有 Dog 调用，会优化为：
// Dog.speak() → 静态调用 → 可能内联
```

这也是为什么 Swift 的性能通常优于 ObjC — **编译器帮你把动态派发优化成静态派发**。

### 7b.7 与 ObjC Runtime 的关系

| 特性               | Swift class    | Swift struct/enum | ObjC class                 |
| ---------------- | -------------- | ----------------- | -------------------------- |
| 默认派发             | vtable         | 静态                | `objc_msgSend`             |
| 运行时方法替换          | ❌ 需要 `dynamic` | ❌                 | ✅ Method Swizzling         |
| 运行时添加方法          | ❌              | ❌                 | ✅ `class_addMethod`        |
| 消息转发             | ❌ 需要 `@objc`   | ❌                 | ✅ 三个救援阶段                   |
| `isKindOfClass:` | ✅ 继承自 NSObject | ❌                 | ✅                          |
| 类的动态创建           | ❌              | ❌                 | ✅ `objc_allocateClassPair` |

> **核心区别**：ObjC 的所有方法调用都走 `objc_msgSend`（动态），而 Swift 的默认选择是静态/vtable（编译时确定）。只有显式标记 `@objc dynamic` 的方法才会回到 ObjC 的动态派发路径。

***

## 八、总结速查

| 概念               | 要点                                                                                        |
| ---------------- | ----------------------------------------------------------------------------------------- |
| Runtime 本质       | C + 汇编实现的动态库，ObjC 动态性的基础                                                                  |
| `objc_msgSend`   | 汇编实现，先查 cache（O(1)），未命中则沿继承链查 methods                                                     |
| 方法缓存             | `cache_t` 哈希表，`SEL >> 哈希位移 → slot`，存 `IMP`                                                |
| 方法未找到            | `+resolveInstanceMethod:` → `forwardingTargetForSelector:` → `forwardInvocation:` → crash |
| Method Swizzling | 交换两个方法的 IMP，用 `+load` + `dispatch_once` + `class_addMethod` 安全实现                          |
| 关联对象             | `objc_setAssociatedObject` / `objc_getAssociatedObject`，给分类加属性                            |
| isa 指针链          | instance → class → meta-class → root meta-class → self                                    |
| Swift 派发         | struct/enum → 静态派发；class → vtable；`@objc dynamic` → `objc_msgSend`                        |
