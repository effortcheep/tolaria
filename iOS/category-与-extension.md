---
type: Note
---
# Category 与 Extension

> **Category（分类）和 Extension（扩展）有什么区别？Category 能添加属性吗？Category 的 load 和 initialize 方法执行时机和顺序是什么？**

---

## 一、Category 和 Extension 的区别

| 对比项 | Category（分类） | Extension（扩展） |
|--------|------------------|-------------------|
| **作用** | 为已有的类添加方法（和协议） | 为类添加私有声明（属性、方法） |
| **能否添加实例变量** | ❌ 不能 | ❌ 不能（但可以在 `.m` 中声明属性，用关联对象实现） |
| **能否添加方法** | ✅ 可以 | ✅ 可以 |
| **是否有独立实现文件** | 通常有（`.h` + `.m`） | 没有，写在主类的 `.m` 文件中 |
| **命名** | `ClassName+CategoryName.h/m` | 匿名 `@interface ClassName ()` |
| **运行时可见性** | 编译时 + 运行时都可见 | 仅编译时可见（编译后合并到主类） |
| **用途** | 模块化、给系统类添加方法 | 声明私有属性/方法，隐藏实现细节 |

### 1.1 Category 的本质

Category 在编译后是一个 `category_t` 结构体，在运行时通过 `runtime` 附加到目标类上：

```c
struct category_t {
    const char *name;           // 分类名
    classref_t cls;             // 目标类
    method_list_t *instanceMethods;  // 实例方法
    method_list_t *classMethods;     // 类方法
    protocol_list_t *protocols;      // 协议
    property_list_t *instanceProperties; // 属性（只有声明，没有 ivar）
};
```

关键流程：
```
编译时：Category 编译成 category_t 结构体
   ↓
运行时：dyld 加载后，runtime 调用 _objc_init → _read_images → attachCategories
   ↓
将 category 的方法列表插入到类的方法列表**前面**（方法查找时优先找到 category 的方法）
```

### 1.2 Extension 的本质

Extension 是编译时的语法糖，编译后其内容直接合并到主类中，不存在独立的结构体。所以 Extension 可以声明属性，但必须在主类的 `@implementation` 中手动合成 ivar 和 getter/setter。

```objc
// Person.m
@interface Person ()
// Extension：声明私有属性
@property (nonatomic, copy) NSString *secret;
@end

@implementation Person
// secret 的 ivar 和 getter/setter 自动合成在这里
@end
```

---

## 二、Category 能添加属性吗？

### 2.1 直接写 @property 会怎样？

```objc
@interface Person (MyCategory)
@property (nonatomic, copy) NSString *nickname;  // ⚠️ 编译不报错，但运行时没有 ivar
@end
```

**结果：**
- 编译器会生成 getter/setter 方法声明
- 但**不会生成 `_nickname` 实例变量**
- 如果你自己写了 getter/setter 实现，可以正常工作
- 如果不写，运行时调用 `person.nickname` 会 crash：`unrecognized selector sent to instance`

### 2.2 用关联对象（Associated Object）实现

这是 Category 添加「属性」的标准做法：

```objc
#import <objc/runtime.h>

@interface Person (MyCategory)
@property (nonatomic, copy) NSString *nickname;
@end

@implementation Person (MyCategory)

- (void)setNickname:(NSString *)nickname {
    objc_setAssociatedObject(self, @selector(nickname), nickname, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)nickname {
    return objc_getAssociatedObject(self, @selector(nickname));
}

@end
```

### 2.3 关联对象的底层原理

```
关联对象存储在一个全局的 AssociationsHashMap 中：

AssociationsHashMap（全局）
└── key: 对象指针（被关联的对象）
    └── ObjectAssociationMap
        └── key: selector（如 @selector(nickname)）
            └── ObjcAssociation（policy + value）

objc_setAssociatedObject 的流程：
1. 创建 ObjcAssociation（policy, value）
2. 以对象指针为 key，在 AssociationsHashMap 中找到/创建 ObjectAssociationMap
3. 以 selector 为 key，存入 ObjcAssociation

对象 dealloc 时：
runtime 调用 _object_remove_assocations，批量清理所有关联对象
```

### 2.4 关联对象的策略

| 策略 | 等价属性修饰符 | 说明 |
|------|---------------|------|
| `OBJC_ASSOCIATION_ASSIGN` | `assign` | 弱引用，不 retain |
| `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | `strong, nonatomic` | 强引用，非原子 |
| `OBJC_ASSOCIATION_COPY_NONATOMIC` | `copy, nonatomic` | 拷贝，非原子 |
| `OBJC_ASSOCIATION_RETAIN` | `strong` | 强引用，原子 |
| `OBJC_ASSOCIATION_COPY` | `copy` | 拷贝，原子 |

---

## 三、Category 的 load 和 initialize

### 3.1 +load 方法

**执行时机：** App 启动时，runtime 加载类和分类的镜像（image）时调用。

```
App 启动
   ↓
dyld 加载 Mach-O
   ↓
runtime 初始化：_objc_init → load_images
   ↓
遍历所有已注册的类：
  按 父类 → 子类 的顺序调用 +load
   ↓
遍历所有分类：
  按分类加载顺序调用 +load
```

**关键特点：**

1. **每个类/分类的 +load 只调用一次**
2. **不会被继承**：子类有 +load 就调子类的，没有就不调（不会调父类的 +load）
3. **不遵循继承链查找**：和普通方法不同，runtime 直接用函数指针调用 `load_method`，不走 `objc_msgSend`
4. **调用顺序**：
   - 类的 +load：父类 → 子类（按编译顺序）
   - 分类的 +load：按编译顺序（Compile Sources 中的顺序）

### 3.2 +initialize 方法

**执行时机：** 第一次收到消息时（懒加载），通过 `objc_msgSend` 触发。

```
代码第一次调用 [Person class] 或 person.name 等
   ↓
objc_msgSend 查找方法
   ↓
发现类的 initialized 标记为 NO
   ↓
先调用 +initialize（通过 objc_msgSend 发送）
   ↓
标记 initialized = YES
   ↓
再执行实际的方法调用
```

**关键特点：**

1. **懒加载**：不用不调，第一次使用类时才调
2. **走消息发送**：子类如果没有实现 +initialize，`objc_msgSend` 会沿继承链找到父类的实现并执行（但每个类的 +initialize 只会执行一次，runtime 有 initialized 标记）
3. **线程安全**：runtime 内部加了锁
4. **调用顺序**：父类 → 子类（因为子类调用前会先触发父类的 initialize）

### 3.3 load 和 initialize 对比

| 对比项 | +load | +initialize |
|--------|-------|-------------|
| **调用时机** | App 启动时，加载镜像 | 第一次收到消息时 |
| **每个类** | 调一次 | 调一次 |
| **每个分类** | 也调一次 | 不额外调（覆盖主类的） |
| **父类** | 父类先调 | 父类先调 |
| **是否继承** | ❌ 不继承 | ✅ 继承（走 objc_msgSend） |
| **调用方式** | 直接函数指针调用 | objc_msgSend |
| **线程** | 主线程 | 第一次消息发送的线程 |
| **子类实现** | 子类有就调子类的，没有就不调 | 子类没有就调父类的 |
| **分类实现** | 分类有就调分类的，不影响主类 | 分类会**覆盖**主类的 initialize |

### 3.4 复杂情况的调用顺序示例

```objc
@interface Animal : NSObject
@end
@interface Dog : Animal
@end
@interface Dog (Swim)
@end
@interface Dog (Bark)
@end
```

**+load 调用顺序：**
```
1. +[Animal load]         // 父类先
2. +[Dog load]            // 子类后
3. +[Dog(Swim) load]      // 分类按编译顺序
4. +[Dog(Bark) load]
```

**+initialize 调用顺序：**
```
// 假设代码中第一次使用 Dog
1. +[Animal initialize]   // 先触发父类（因为走消息发送）
2. +[Dog(Bark) initialize] // 分类的 initialize 覆盖了主类的
// 注意：如果两个分类都有 initialize，只调用后加载的那个
```

**为什么父类的 +initialize 还会被调用？**

「分类覆盖主类」只针对**同一个类**，不影响父类。两次调用是独立的 `objc_msgSend`：

```
代码第一次使用 Dog
    ↓
objc_msgSend(Dog, @selector(initialize))
    ↓
runtime 发现 Dog 没有 initialized
    ↓
runtime 先确保父类已 initialized
    ↓
objc_msgSend(Animal, @selector(initialize))  ← 发给 Animal，独立调用
    ↓
Animal 没有分类，执行 Animal 自己的 +initialize ✅
    ↓
标记 Animal initialized = YES
    ↓
回到 Dog，objc_msgSend(Dog, @selector(initialize))  ← 发给 Dog
    ↓
Dog(Bark) 有 +initialize，覆盖了 Dog 主类的
    ↓
执行 +[Dog(Bark) initialize] ✅（Dog 主类的 +initialize 不执行）
    ↓
标记 Dog initialized = YES
```

关键点：
- `+[Animal initialize]` 是发给 `Animal` 类的消息，`+[Dog(Bark) initialize]` 是发给 `Dog` 类的消息
- 这是**两次独立的 `objc_msgSend`**，不是同一个方法的继承覆盖
- 「分类覆盖」只发生在 Dog 内部：`Dog(Bark)` 的 `+initialize` 覆盖了 `Dog` 的 `+initialize`
- `Animal` 是另一个类，它的 `+initialize` 独立执行，不受 `Dog` 分类的影响

### 3.5 常见坑

```objc
// ❌ 坑1：initialize 被分类覆盖
// Dog.m
@implementation Dog
+ (void)initialize {
    NSLog(@"Dog initialize");  // 不会执行！被分类覆盖了
}
@end

// Dog+Bark.m
@implementation Dog (Bark)
+ (void)initialize {
    NSLog(@"Dog(Bark) initialize");  // 实际执行的是这个
}
@end

// ✅ 正确做法：用 class 判断
+ (void)initialize {
    if (self == [Dog class]) {  // 只在本类时执行
        NSLog(@"Dog initialize");
    }
}

// ❌ 坑2：initialize 中调用了子类的方法
// 因为 initialize 会被继承，如果子类没有重写，
// 父类的 initialize 会在子类的上下文中执行，
// 此时 self 是子类，调用的方法可能不存在或行为不符预期
```

---

## 四、Category 方法与主类方法同名会怎样？

```objc
// Person.m
@implementation Person
- (void)greet {
    NSLog(@"Hello from Person");
}
@end

// Person+Category.m
@implementation Person (Category)
- (void)greet {
    NSLog(@"Hello from Category");
}
@end

// 调用
[person greet]; // → "Hello from Category"
```

**原因：** Category 的方法列表被插入到类的方法列表**前面**，`objc_msgSend` 查找方法时先找到 Category 的。

**风险：** 如果两个分类有同名方法，调用顺序取决于编译顺序（Compile Sources 中的顺序），这是**未定义行为**，应避免。

---

## 五、总结速查表

| 概念 | 要点 |
|------|------|
| **Category** | 运行时添加方法到已有类，不能添加 ivar |
| **Extension** | 编译时语法糖，声明私有属性/方法，编译后合并到主类 |
| **Category 添加属性** | 必须用 `objc_setAssociatedObject` + `objc_getAssociatedObject` |
| **关联对象存储** | 全局 AssociationsHashMap，dealloc 时自动清理 |
| **+load** | 启动时调用，不继承，直接函数指针调用，分类不影响主类 |
| **+initialize** | 首次消息时调用，继承（走 objc_msgSend），分类会覆盖主类 |
| **同名方法** | Category 覆盖主类（方法插入前面），多个分类间取决于编译顺序 |
| **+initialize 安全写法** | `if (self == [ClassName class])` 防止被子类/分类覆盖 |
