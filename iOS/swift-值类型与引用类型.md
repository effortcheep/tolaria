---
type: review
---
# Swift 值类型与引用类型

> **Swift 中 struct（值类型）和 class（引用类型）有什么区别？为什么 Swift 标准库大量使用 struct？什么时候应该用 class 而不是 struct？**

---

## 一、值类型 vs 引用类型：核心区别

| 特性 | 值类型（struct / enum） | 引用类型（class） |
|------|------------------------|------------------|
| **赋值** | 拷贝一份独立副本 | 共享同一份数据（指针拷贝） |
| **存储位置** | 栈上（大对象可能优化到堆） | 堆上 |
| **内存管理** | 自动，离开作用域销毁 | ARC 引用计数 |
| **线程安全** | 天然安全（各自独立副本） | 需要手动同步 |
| **继承** | ❌ 不能继承（只能协议扩展） | ✅ 支持继承 |
| **引用计数** | ❌ 无 | ✅ 有 |
| **identity** | 无概念（值相等就行） | 有 `===` 判断是否同一对象 |

```swift
// 值类型：赋值 = 拷贝
struct Point {
    var x: Int
    var y: Int
}
var a = Point(x: 1, y: 2)
var b = a        // b 是 a 的独立副本
b.x = 10
print(a.x)       // 1（a 不受影响）

// 引用类型：赋值 = 共享
class Person {
    var name: String
    init(name: String) { self.name = name }
}
let p1 = Person(name: "Tom")
let p2 = p1      // p2 和 p1 指向同一对象
p2.name = "Jerry"
print(p1.name)    // "Jerry"（p1 也变了！）
```

---

## 二、值类型的 Copy-on-Write（COW）

Swift 的值类型不是每次都真正拷贝，而是用了**写时复制**优化：

```swift
var a = [1, 2, 3]   // a 的引用计数 = 1
var b = a            // 不拷贝！a 和 b 共享底层 buffer，引用计数 = 2
b.append(4)          // b 要修改，此时才真正拷贝一份（引用计数各自为 1）
```

底层实现：

```
┌─────────┐     ┌──────────────────┐
│ struct  │────→│  Heap Buffer     │
│ Array   │     │  [1, 2, 3]       │
│ (isa)   │     │  refCount = 2    │
└─────────┘     └──────────────────┘
                       ↑
┌─────────┐           │
│ struct  │───────────┘
│ Array   │
│ (b)     │
└─────────┘

// b.append(4) 时：
// 1. 检查 refCount > 1 → 需要拷贝
// 2. 创建新 buffer，复制元素
// 3. b 指向新 buffer
// 4. 原 buffer refCount - 1
```

Swift 标准库的 `Array`、`Dictionary`、`Set`、`String` 都实现了 COW，所以虽然是值类型语义，但性能接近引用类型。

---

## 三、为什么 Swift 标准库大量使用 struct？

### 3.1 安全性

```swift
// 用 struct，不会意外被其他代码修改
func process(point: Point) {
    // 这里修改 point 不会影响调用方的原始值
    // 因为 point 是拷贝
}

// 用 class，可能被意外修改
func process(person: Person) {
    // 这里修改 person 会影响调用方的原始对象
    // 因为 person 是引用
}
```

### 3.2 性能

```
struct：栈分配，无引用计数开销，无堆分配开销
class：堆分配 + 引用计数（alloc → retain → release → dealloc）
```

对于 `Int`、`Double`、`Bool`、`String`、`Array` 这些高频使用的类型，struct 的栈分配和无引用计数开销带来了显著的性能提升。

### 3.3 值语义更符合直觉

```swift
var s1 = "hello"
var s2 = s1
s2.append(" world")
print(s1)  // "hello"（符合直觉）

var a1 = [1, 2, 3]
var a2 = a1
a2.append(4)
print(a1)  // [1, 2, 3]（符合直觉）
```

### 3.4 线程安全

```
struct 天然线程安全：
- 每个变量都是独立副本
- 不需要加锁
- 不会有数据竞争

class 需要考虑：
- 多线程同时修改 → 数据竞争
- 需要加锁或使用 DispatchQueue
```

---

## 四、什么时候应该用 class？

### 4.1 需要继承

```swift
// ❌ struct 不能继承
struct Animal {
    func speak() { print("...") }
}
struct Dog: Animal { }  // 编译错误

// ✅ class 可以继承
class Animal {
    func speak() { print("...") }
}
class Dog: Animal {
    override func speak() { print("汪!") }
}
```

### 4.2 需要引用语义（共享状态）

```swift
// 多个地方共享同一个对象，修改一处全部可见
class UserManager {
    static let shared = UserManager()
    var currentUser: User?
}

// 多个 VC 都引用同一个 manager，修改 currentUser 其他 VC 也能看到
```

### 4.3 需要析构（deinit）

```swift
class Resource {
    deinit {
        // 释放资源：关闭文件、断开连接、移除通知等
        print("资源释放")
    }
}

// struct 没有 deinit
```

### 4.4 需要引用计数 / 弱引用

```swift
// delegate 模式需要 weak 打破循环引用
protocol ViewControllerDelegate: AnyObject {
    func didTapButton()
}

class MyViewController: UIViewController {
    weak var delegate: ViewControllerDelegate?  // weak 只能用于 class
}

// struct 没有引用计数，不能 weak
```

### 4.5 需要 ObjC 互操作

```swift
// 要暴露给 ObjC，必须用 class
@objc class MyObject: NSObject {
    @objc func doSomething() { }
}

// struct 不能继承 NSObject，不能 @objc
```

---

## 五、决策树：该用 struct 还是 class？

```
需要继承？ → Yes → 用 class
    ↓ No
需要 deinit？ → Yes → 用 class
    ↓ No
需要 weak / 引用计数？ → Yes → 用 class
    ↓ No
需要 ObjC 互操作？ → Yes → 用 class
    ↓ No
需要共享可变状态？ → Yes → 用 class
    ↓ No
用 struct ✅
```

---

## 六、协议中的 Any Object 约束

```swift
// 只能被 class 遵循的协议
protocol SomeDelegate: AnyObject {
    func didUpdate()
}

// 这样 delegate 属性才能用 weak
class ViewController {
    weak var delegate: SomeDelegate?
}

// struct 不能遵循 AnyObject 协议
struct BadDelegate: SomeDelegate { }  // 编译错误
```

---

## 七、Swift 中的常见值类型和引用类型

| 值类型 | 引用类型 |
|--------|---------|
| `Int`, `Double`, `Bool`, `String` | `class` |
| `Array`, `Dictionary`, `Set` | `NSError` |
| `struct`（包括 `Range`, `ClosedRange`） | `NSRegularExpression` |
| `enum`（包括 `Optional`） | `NSCache` |
| `Tuple` | `NotificationCenter` |
| `CharacterSet`, `URLComponents` | `URLSession` |

> ⚠️ 注意：Swift 的 `Array`、`String` 等虽然是值类型，但底层用了 COW，性能接近引用类型。

---

## 八、值类型的陷阱

### 8.1 闭包捕获值类型

```swift
struct User {
    var name: String
    var age: Int
}

// ✅ 值拷贝：赋值 = 独立副本
var user = User(name: "张三", age: 25)
var copy = user           // 拷贝了一份
copy.age = 26
print(user.age)           // 25（各自独立）
print(copy.age)           // 26

// ✅ 闭包捕获 var = 引用捕获，会修改原始值
var user2 = User(name: "李四", age: 30)
let closure = { user2.age = 26 }
closure()
print(user2.age)          // 26（引用捕获，修改了原始值）

// ✅ 用 inout：显式修改原始值，意图更清晰
func updateUser(_ user: inout User) {
    user.age = 26
}
var user3 = User(name: "王五", age: 35)
updateUser(&user3)
print(user3.age)          // 26
```

> **Swift 闭包捕获机制**：闭包捕获 `var` 值类型变量时，编译器会将其**包装为引用类型对象**（heap allocation），闭包持有该对象的引用。所以闭包和外部变量共享同一个存储，闭包内修改**会**影响外部变量。这和 ObjC block 不同——block 捕获值类型时确实是拷贝副本。想要显式表达「修改原始值」的意图，用 `inout` 更清晰，不会有隐式共享的歧义。

### 8.2 struct 中的 class 属性

```swift
struct ViewModel {
    var name: String         // 值类型
    var image: UIImage       // 引用类型！
}

var vm1 = ViewModel(name: "A", image: someImage)
var vm2 = vm1
vm2.name = "B"
vm2.image = anotherImage

print(vm1.name)   // "A"（值类型，独立副本）
print(vm1.image === vm2.image)  // false（image 被重新赋值了）
// 但如果只修改 image 的内部属性，vm1 也会受影响
```

**结论**：struct 里包含 class 属性时，值语义只保护到 struct 层级，class 属性内部仍然是共享的。

---

## 九、总结速查表

| 概念 | 要点 |
|------|------|
| **值类型** | 赋值 = 拷贝，栈分配，无引用计数，天然线程安全 |
| **引用类型** | 赋值 = 共享，堆分配，ARC 管理，需要考虑线程安全 |
| **COW** | 写时复制，Array/String/Dict/Set 都有，值类型性能接近引用类型 |
| **用 struct** | 不需要继承/deinit/weak/ObjC 互操作时 |
| **用 class** | 需要继承/deinit/weak/共享状态/ObjC 互操作时 |
| **AnyObject 协议** | 限制只能 class 遵循，delegate 常用 |
| **struct + class 属性** | 值语义只到 struct 层级，class 属性内部仍共享 |
| **闭包捕获值类型** | Swift 闭包捕获 `var` 值类型 = 引用捕获（会修改原始值），ObjC block = 值拷贝（不会修改原始值） |
| **Swift 标准库** | 大部分是 struct，通过 COW 兼顾安全性和性能 |
