---
type: review
---
# KVO 与 KVC

> **KVC（Key-Value Coding）是什么？KVO（Key-Value Observing）是什么？KVO 的底层实现原理是什么？有什么常见的问题和坑？**

## 一、KVC（Key-Value Coding）

### 1.1 KVC 是什么

KVC 是 Apple 提供的一套**基于字符串的间接访问属性**的机制，定义在 `NSKeyValueCoding` 协议中，`NSObject` 默认实现了它。

```objc
// 传统方式
person.name = @"Tom";
NSString *name = person.name;

// KVC 方式
[person setValue:@"Tom" forKey:@"name"];
NSString *name = [person valueForKey:@"name"];
```

### 1.2 KVC 的查找顺序

**`setValue:forKey:`** 设 key = `age`：

```
1. 查找 setter 方法：setAge: → _setAge:
   ↓ 都没找到
2. 访问 +accessInstanceVariablesDirectly（默认返回 YES）
   → 返回 YES：按 _age, _isAge, age, isAge 顺序找成员变量
   → 返回 NO：跳过，直接到第 3 步
3. 都找不到 → setValue:forUndefinedKey: → 默认抛异常 💥
```

**`valueForKey:`** 设 key = `age`：

```
1. 查找 getter 方法：getAge → age → isAge → _age
   ↓ 都没找到
2. 访问 +accessInstanceVariablesDirectly（默认返回 YES）
   → 返回 YES：按 _age, _isAge, age, isAge 顺序找成员变量
   → 返回 NO：跳过，直接到第 3 步
3. 都找不到 → valueForUndefinedKey: → 默认抛异常 💥
```

> **关键点**：`+accessInstanceVariablesDirectly` 默认返回 `YES`，重写返回 `NO` 可以禁止 KVC 直接访问成员变量，强制走 `xxxForUndefinedKey:`。

### 1.3 KVC 的常见用法

```objc
// 1. 访问私有属性（绕过 getter/setter）
UITextField *textField = ...;
[textField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor"];

// 2. 通过 keyPath 访问嵌套属性
NSString *city = [person valueForKeyPath:@"address.city"];

// 3. 集合运算符
NSArray *ages = [persons valueForKeyPath:@"@avg.age"];
NSNumber *max = [persons valueForKeyPath:@"@max.age"];
NSArray *names = [persons valueForKeyPath:@"@unionOfObjects.name"];

// 4. 字典转模型
NSDictionary *dict = @{@"name": @"Tom", @"age": @25};
Person *person = [[Person alloc] init];
[person setValuesForKeysWithDictionary:dict];

// 5. 模型转字典
NSDictionary *dict = [person dictionaryWithValuesForKeys:@[@"name", @"age"]];
```

---

## 二、KVO（Key-Value Observing）

### 2.1 KVO 是什么

KVO 是基于 KVC 的**观察者模式**实现，当被观察对象的属性值发生变化时，观察者会收到通知。

```objc
// 注册观察者
[person addObserver:self
         forKeyPath:@"name"
            options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
            context:nil];

// 接收通知
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                       context:(void *)context {
    NSLog(@"%@ changed: %@", keyPath, change);
}

// 移除观察者（必须！）
[person removeObserver:self forKeyPath:@"name"];
```

### 2.2 KVO 的使用注意事项

```objc
// 1. 必须成对注册和移除，否则 crash
//    - 重复移除会 crash
//    - 不移除，对象 dealloc 后观察者收到通知会 crash

// 2. 必须在正确线程移除
//    - 在哪个线程注册，最好在哪个线程移除

// 3. 使用 context 指针区分不同观察者
static void *PersonNameContext = &PersonNameContext;
[person addObserver:self forKeyPath:@"name" options:... context:PersonNameContext];

// 4. KVO 触发方式
//    - 自动触发：setter 赋值会自动触发
//    - 手动触发：重写 +automaticallyNotifiesObserversForKey: 返回 NO，
//               然后手动调用 willChangeValueForKey: / didChangeValueForKey:

// 5. 依赖属性
+ (NSSet<NSString *> *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"firstName", @"lastName", nil];
}
```

---

## 三、KVO 的底层实现原理

### 3.1 ISA Swizzling（核心机制）

```
注册 KVO 前：
person.isa → Person 类

注册 KVO 后：
person.isa → NSKVONotifying_Person 类（Runtime 动态创建的子类）
                  ↑
                  继承自 Person
```

### 3.2 动态子类的结构

```
NSKVONotifying_Person
├── -setAge:          ← 重写了 setter，内部调用 willChangeValueForKey:
│                         → 原始 setter → didChangeValueForKey:
│                         → 通知观察者
├── -class            ← 返回 "Person"（伪装）
├── -dealloc          ← 自动清理 KVO 注册
└── -_isKVOA          ← 标记这是一个 KVO 动态子类
```

### 3.3 完整触发流程

```
1. 调用 [person addObserver:self forKeyPath:@"age" ...]
   ↓
2. Runtime 动态创建 NSKVONotifying_Person 子类
   ↓
3. 重写 -setAge: 方法
   ↓
4. 修改 person.isa 指向新子类
   ↓
5. 代码执行 person.age = 25
   ↓
6. 实际调用 NSKVONotifying_Person 的 -setAge:
   ↓
7. 内部执行：
   - [self willChangeValueForKey:@"age"]
   - [super setAge:25]    // 调用原始 setter
   - [self didChangeValueForKey:@"age"]
   ↓
8. didChangeValueForKey: 通知所有观察者
   ↓
9. 观察者的 observeValueForKeyPath: 被调用
```

### 3.4 验证 KVO 子类存在

```objc
// 注册 KVO 前
NSLog(@"类名: %@", object_getClass(person));  // Person

// 注册 KVO 后
NSLog(@"类名: %@", object_getClass(person));  // NSKVONotifying_Person
NSLog(@"superclass: %@", class_getSuperclass(object_getClass(person)));  // Person

// class 方法被重写，返回原始类
NSLog(@"class: %@", [person class]);  // Person（不是 NSKVONotifying_Person）
```

---

## 四、手动触发 KVO

```objc
// 1. 关闭自动触发
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO;
    }
    return [super automaticallyNotifiesObserversForKey:key];
}

// 2. 手动触发
- (void)setName:(NSString *)name {
    [self willChangeValueForKey:@"name"];
    _name = name;
    [self didChangeValueForKey:@"name"];
}

// 3. 依赖属性（fullName 依赖 firstName 和 lastName）
+ (NSSet<NSString *> *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"firstName", @"lastName", nil];
}
```

---

## 五、常见的问题和坑

### 5.1 Crash：未移除观察者

```objc
// ❌ 崩溃场景
// ViewController 注册了 KVO，但在 dealloc 中忘记 removeObserver
// 被观察对象释放后，通知发给了野指针 → crash

// ✅ 正确做法
- (void)dealloc {
    [self.person removeObserver:self forKeyPath:@"name"];
}

// 或者使用 KVOController（Facebook 开源库，自动管理生命周期）
[self.KVOController observe:self.person keyPath:@"name" options:... block:^(id observer, id object, NSDictionary *change) {
    // 处理变化
}];
```

### 5.2 Crash：重复移除

```objc
// ❌ 移除两次 → crash
[person removeObserver:self forKeyPath:@"name"];
[person removeObserver:self forKeyPath:@"name"];  // 💥

// ✅ 使用 @try catch 或标记已移除
```

### 5.3 通过 ivar 直接赋值不触发 KVO

```objc
// ❌ 不触发 KVO（绕过了 setter，直接改 ivar）
_person->_name = @"Tom";                    // 直接访问 ivar
[person setValue:@"Tom" forKey:@"_name"];    // forKey:@"_name" 也是直接写 ivar

// ✅ 触发 KVO（通过 setter 或 KVC）
person.name = @"Tom";                       // 走 setter，自动触发 KVO
[person setValue:@"Tom" forKey:@"name"];    // forKey:@"name" 走 setter，触发 KVO
```

### 5.4 子类化问题

```objc
// ❌ 如果 Person 重写了 -setAge: 但没有调用 will/didChangeValueForKey:
// KVO 就无法触发

// ✅ 确保自定义 setter 调用了 super 或手动触发 will/did
- (void)setAge:(NSInteger)age {
    [self willChangeValueForKey:@"age"];
    _age = age;
    [self didChangeValueForKey:@"age"];
}
```

### 5.5 多线程问题

```objc
// ❌ 在子线程注册，主线程移除，可能导致问题

// ✅ KVO 的通知在触发的线程同步回调，注意线程安全
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if ([NSThread isMainThread]) {
        [self handleKVO:keyPath change:change];
    } else {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self handleKVO:keyPath change:change];
        });
    }
}
```

### 5.6 性能问题

```objc
// ❌ 高频属性用 KVO 可能导致性能问题
// 比如 scrollView.contentOffset 用 KVO 监听 → 卡顿

// ✅ 优先使用 delegate（scrollViewDidScroll:）而不是 KVO
```

---

## 六、KVO 的替代方案

### 6.1 Swift 中的观察

```swift
// Swift 4+ 使用 @Published（Combine）
class Person: ObservableObject {
    @Published var name: String = ""
}

// SwiftUI 中自动观察
struct ContentView: View {
    @ObservedObject var person: Person

    var body: some View {
        Text(person.name)  // name 变化时自动更新 UI
    }
}

// 或者直接用 willSet/didSet
class Person {
    var name: String = "" {
        willSet { print("即将变为 \(newValue)") }
        didSet  { print("从 \(oldValue) 变为 \(name)") }
    }
}
```

### 6.2 第三方库

| 库 | 特点 |
|---|------|
| **KVOController**（Facebook） | 自动管理生命周期，block 回调，线程安全 |
| **ReactiveCocoa** | 函数响应式编程，KVO 只是其中一部分 |
| **RxSwift** | 响应式编程，`observe()` 替代 KVO |
| **Combine** | Apple 官方响应式框架，`publisher(for:)` 替代 KVO |

---

## 七、总结速查表

| 概念 | 要点 |
|------|------|
| **KVC** | 基于字符串的属性访问，查找顺序：setter → `accessInstanceVariablesDirectly` → ivar → undefinedKey |
| **KVO** | 基于 KVC 的观察者模式，属性变化时通知观察者 |
| **底层原理** | Runtime 动态创建 `NSKVONotifying_` 子类，重写 setter，修改 isa 指针 |
| **触发条件** | 必须通过 setter 赋值，直接写 ivar 不触发 |
| **必须 removeObserver** | 否则被观察对象释放后 crash |
| **手动触发** | 重写 `automaticallyNotifiesObserversForKey:` 返回 NO，手动调用 will/did |
| **依赖属性** | `keyPathsForValuesAffectingXxx` 声明依赖 |
| **Swift 替代** | `@Published` + `ObservableObject`，或 `willSet/didSet` |
| **第三方推荐** | KVOController（自动生命周期管理） |
