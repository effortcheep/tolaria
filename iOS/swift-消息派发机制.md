---
type: review
related_to: "[[Runtime]]"
---

# Swift 的消息派发机制

Swift 作为一门同时支持面向对象和面向协议的语言，提供了三种消息派发（方法调用）机制：**静态派发**、**表派发（虚函数表）**、**消息派发（objc_msgSend）**。

---

## 1. 三种派发方式

### 1.1 静态派发（Static Dispatch）

- **编译期确定**调用地址，直接跳转到函数实现
- **最快**的派发方式，无运行时开销
- 编译器可以进行**内联优化**
- 适用场景：
  - `struct`、`enum` 的方法
  - `final` 修饰的类和方法
  - `private` 方法（编译器可推断无子类覆盖）
  - 通过 `@inlinable` 标记的函数

```swift
struct Point {
    func distance() -> Double { ... }  // 静态派发
}

class Animal {
    final func breathe() { ... }  // 静态派发
}
```

### 1.2 表派发（Table Dispatch / Virtual Table）

- 类似 C++ 的虚函数表（vtable）
- **运行时**通过函数指针表查找目标方法
- 性能介于静态派发和消息派发之间
- 适用场景：
  - `class` 的普通实例方法（默认派发方式）
  - `protocol` extension 中满足协议的方法

```swift
class Animal {
    func speak() { ... }  // 表派发（vtable）
}

// vtable 结构示意：
// ┌──────────────┬──────────────────────┐
// │ 类/方法       │ 函数指针              │
// ├──────────────┼──────────────────────┤
// │ Animal.speak │ 0x1000A              │
// │ Dog.speak    │ 0x1000B (override)   │
// └──────────────┴──────────────────────┘
```

### 1.3 消息派发（Message Dispatch / objc_msgSend）

- OC Runtime 的消息机制，支持**动态方法解析和转发**
- **最灵活**但**最慢**的派发方式
- 适用场景：
  - 继承自 `NSObject` 的类
  - 使用 `@objc` 标记的方法
  - 使用 `dynamic` 修饰的方法

```swift
class MyObject: NSObject {
    @objc dynamic func doSomething() { ... }  // 消息派发
}
```

---

## 2. 派发方式对比

| 特性 | 静态派发 | 表派发 | 消息派发 |
|------|---------|--------|---------|
| 确定时机 | 编译期 | 运行时 | 运行时 |
| 性能 | ⚡️ 最快 | 🔵 中等 | 🐢 最慢 |
| 灵活性 | 无多态 | 支持多态 | 完全动态 |
| 可内联 | ✅ | ❌ | ❌ |
| 方法交换 | ❌ | ❌ | ✅ |

---

## 3. 关键修饰符对派发的影响

| 修饰符 | 效果 |
|--------|------|
| `final` | 强制静态派发，禁止子类覆盖 |
| `dynamic` | 强制消息派发（objc_msgSend） |
| `@objc` | 暴露给 OC 运行时，使用消息派发 |
| `@objc dynamic` | 最完整的 OC 兼容，支持 KVO 等 |
| `private` | 编译器可能优化为静态派发 |
| `@inlinable` | 允许跨模块内联 |

---

## 4. 协议的派发方式

### 4.1 协议本身作为类型使用 → 表派发（witness table）

```swift
protocol Drawable {
    func draw()
}

func render(_ item: Drawable) {
    item.draw()  // 通过 witness table 派发
}
```

### 4.2 协议 extension 的陷阱

```swift
protocol Greetable {
    func greet()
}

extension Greetable {
    func greet() { print("Hello") }  // 默认实现
}

class Person: Greetable {}
// Person().greet() — 调用的是 extension 的实现
// 但如果 Person 自己实现了 greet()，则走表派发
```

> ⚠️ 协议 extension 中的方法默认是**静态派发**，除非该方法是协议本身声明的（此时走 witness table）。

---

## 5. @_silgen_name 与派发方式

```swift
// 可以查看编译器生成的派发方式
// 使用 swiftc -emit-sil 查看中间 SIL 代码
// %0 = class_method %x : $MyClass, #MyClass.method : (MyClass) -> () -> ()
// vs
// %0 = function_ref @MyClass.method : $@convention(method) (@guaranteed MyClass) -> ()
```

---

## 6. 性能优化建议

1. **优先使用 struct/enum** → 天然静态派发
2. **类方法标记 `final`** → 如果不需要子类覆盖
3. **避免不必要的 `dynamic`** → 除非需要 KVO 或运行时方法交换
4. **理解协议 extension 的派发行为** → 避免意外的静态派发
5. **使用 `@inlinable` 跨模块优化** → 框架 API 性能优化

---

## 7. 常见面试问题

**Q: Swift 中 class 的方法默认是什么派发方式？**
A: 表派发（vtable），不同于 OC 的消息派发。

**Q: `@objc` 和 `dynamic` 的区别？**
A: `@objc` 只是暴露给 OC 运行时，仍可能是表派发；`dynamic` 强制使用消息派发。`@objc dynamic` 组合才能完整使用 OC 运行时特性。

**Q: 为什么 protocol extension 的方法调用结果有时"不符合预期"？**
A: 协议 extension 中的实现是静态派发的，当变量类型是协议类型时，编译器在编译期就确定了调用地址，不会走到子类的实现。

**Q: Swift 的方法派发比 OC 快吗？**
A: 大多数情况下更快，因为 Swift 优先使用静态派发和表派发，避免了 objc_msgSend 的哈希查找开销。但灵活性也因此降低。
