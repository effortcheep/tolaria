---
type: review
---
# Copy 与 MutableCopy

> `copy` **和** `mutableCopy` **的区别是什么？深拷贝和浅拷贝的区别是什么？分别对 NSString、NSArray、NSMutableDictionary 调用 copy 和 mutableCopy，得到的是什么？为什么 NSString 属性要用 copy 修饰？**

## 一、copy 和 mutableCopy 的区别

| | `copy` | `mutableCopy` |
|--|--------|---------------|
| **返回类型** | 不可变对象（immutable） | 可变对象（mutable） |
| **是否产生新对象** | 不一定（见下方规则） | 一定产生新对象 |
| **典型返回类型** | `NSString`、`NSArray`、`NSDictionary` | `NSMutableString`、`NSMutableArray`、`NSMutableDictionary` |

简单记忆：
- `copy` → 产出不可变的副本
- `mutableCopy` → 产出可变的副本

---

## 二、深拷贝与浅拷贝

| | 浅拷贝（Shallow Copy） | 深拷贝（Deep Copy） |
|--|------------------------|---------------------|
| **是否复制对象本身** | 是 | 是 |
| **是否复制内部元素** | ❌ 不复制，共享引用 | ✅ 逐个复制 |
| **新对象指针** | 可能相同（不可变 → 不可变时） | 一定不同 |
| **修改内部元素** | 原对象也会受影响 | 原对象不受影响 |

```
浅拷贝：
  原数组 [A, B, C]  ──┐
                       ├─→ 共享 A, B, C 对象
  新数组 [A, B, C]  ──┘

深拷贝：
  原数组 [A, B, C]
  新数组 [A', B', C']   ← 元素也是新拷贝的
```

⚠️ **Foundation 的 `copy` 大部分是浅拷贝**（只复制容器，不复制元素）。要实现真正的深拷贝需要用 `initWithArray:copyItems:YES` 或 `NSKeyedArchiver`。

---

## 三、各类型调用 copy / mutableCopy 的结果

### 3.1 NSString

| 操作 | 源类型 | 返回类型 | 是否新对象 | 原因 |
|------|--------|---------|-----------|------|
| `copy` | `NSString`（不可变） | `NSString` | ❌ 不是（指向自身） | 不可变 copy 不可变 → 没必要复制，返回自身 |
| `mutableCopy` | `NSString`（不可变） | `NSMutableString` | ✅ 是 | 产出可变副本，一定是新对象 |

```objc
NSString *str = @"Hello";
NSString *strCopy = [str copy];             // strCopy == str（同一指针）
NSMutableString *strMut = [str mutableCopy]; // strMut != str（新对象）
```

### 3.2 NSMutableString

| 操作 | 源类型 | 返回类型 | 是否新对象 | 原因 |
|------|--------|---------|-----------|------|
| `copy` | `NSMutableString`（可变） | `NSString` | ✅ 是 | 可变 → 不可变，必须生成新对象 |
| `mutableCopy` | `NSMutableString`（可变） | `NSMutableString` | ✅ 是 | 可变 → 可变，必须生成新对象（互不影响） |

```objc
NSMutableString *mStr = [NSMutableString stringWithString:@"Hello"];
NSString *mStrCopy = [mStr copy];                   // 新的不可变字符串
NSMutableString *mStrMut = [mStr mutableCopy];      // 新的可变字符串
[mStr appendString:@" World"];                       // 只影响原对象
```

### 3.3 NSArray

| 操作 | 源类型 | 返回类型 | 是否新对象 | 是否拷贝元素 |
|------|--------|---------|-----------|-------------|
| `copy` | `NSArray`（不可变） | `NSArray` | ❌ 不是 | — |
| `mutableCopy` | `NSArray`（不可变） | `NSMutableArray` | ✅ 是 | ❌ 浅拷贝，元素共享 |

```objc
NSArray *arr = @[@"A", @"B"];
NSArray *arrCopy = [arr copy];              // arrCopy == arr（同一指针）
NSMutableArray *arrMut = [arr mutableCopy];  // 新数组，但元素 A、B 是共享的
```

### 3.4 NSMutableArray

| 操作 | 源类型 | 返回类型 | 是否新对象 | 是否拷贝元素 |
|------|--------|---------|-----------|-------------|
| `copy` | `NSMutableArray`（可变） | `NSArray` | ✅ 是 | ❌ 浅拷贝 |
| `mutableCopy` | `NSMutableArray`（可变） | `NSMutableArray` | ✅ 是 | ❌ 浅拷贝 |

```objc
NSMutableArray *mArr = [NSMutableArray arrayWithObjects:mutStr, @"B", nil];
NSArray *mArrCopy = [mArr copy];            // 新的不可变数组，元素共享
NSMutableArray *mArrMut = [mArr mutableCopy]; // 新的可变数组，元素共享
```

### 3.5 NSDictionary / NSMutableDictionary

规则与 NSArray 完全一致：

| 操作 | 源类型 | 返回类型 | 是否新对象 | 是否拷贝元素 |
|------|--------|---------|-----------|-------------|
| `copy` | `NSDictionary`（不可变） | `NSDictionary` | ❌ 不是 | — |
| `mutableCopy` | `NSDictionary`（不可变） | `NSMutableDictionary` | ✅ 是 | ❌ 浅拷贝 |
| `copy` | `NSMutableDictionary`（可变） | `NSDictionary` | ✅ 是 | ❌ 浅拷贝 |
| `mutableCopy` | `NSMutableDictionary`（可变） | `NSMutableDictionary` | ✅ 是 | ❌ 浅拷贝 |

### 3.6 规则总结

```
                    源对象
               ┌───────────────────┐
               │                   │
          不可变（immutable）   可变（mutable）
               │                   │
         ┌─────┴─────┐       ┌─────┴─────┐
       copy      mutableCopy  copy    mutableCopy
         │           │        │          │
     不新对象     新对象    新对象     新对象
    （返回自身） （可变）  （不可变）  （可变）
```

**一句话规则：**
- **不可变 copy 不可变** → 不产生新对象（因为没必要）
- **其他三种情况** → 一定产生新对象
- **mutableCopy 一定产生新对象**
- **容器的 copy/mutableCopy 都是浅拷贝**（不复制元素）

---

## 四、为什么 NSString 属性要用 copy 修饰？

### 4.1 问题场景

```objc
// .h
@property (nonatomic, strong) NSString *name;

// 使用
NSMutableString *mutStr = [NSMutableString stringWithFormat:@"Tom"];
person.name = mutStr;  // mutStr 和 person.name 指向同一对象

[mutStr appendString:@" and Jerry"];
NSLog(@"%@", person.name);  // "Tom and Jerry" ← 被意外修改了！
```

`strong` 只是引用计数 +1，不会复制对象。`mutStr` 和 `person.name` 指向**同一个可变对象**，外部修改 `mutStr` 会影响 `person.name`。

### 4.2 用 copy 解决

```objc
@property (nonatomic, copy) NSString *name;

NSMutableString *mutStr = [NSMutableString stringWithFormat:@"Tom"];
person.name = mutStr;  // 内部调用 [mutStr copy] → 生成新的不可变 NSString

[mutStr appendString:@" and Jerry"];
NSLog(@"%@", person.name);  // "Tom" ← 不受影响 ✅
```

`copy` 修饰符会让 setter 内部调用 `[newValue copy]`，生成一个**不可变的副本**存储，外部修改原字符串不会影响属性值。

### 4.3 为什么不用 mutableCopy

```objc
// ❌ 如果用 mutableCopy
@property (nonatomic, copy) NSString *name;
// setter 内部: self.name = [newValue mutableCopy];
// 返回 NSMutableString，但声明是 NSString，类型不一致
// 而且失去了不可变的安全性
```

### 4.4 哪些属性该用 copy

| 属性类型 | 推荐修饰符 | 原因 |
|---------|-----------|------|
| `NSString` | `copy` | 防止外部 NSMutableString 意外修改 |
| `NSArray` | `copy` | 防止外部 NSMutableArray 意外修改 |
| `NSDictionary` | `copy` | 防止外部 NSMutableDictionary 意外修改 |
| `NSSet` | `copy` | 防止外部 NSMutableSet 意外修改 |
| `Block` | `copy` | 将栈上的 Block 拷贝到堆上（ARC 下系统自动处理） |
| 其他对象 | `strong` | 默认引用计数管理 |

### 4.5 copy 属性的 setter 实质

```objc
// 编译器自动生成的 setter（copy 修饰）
- (void)setName:(NSString *)name {
    _name = [name copy];  // 调用 copy 协议方法
}
```

---

## 五、真正的深拷贝方案

Foundation 的 `copy` / `mutableCopy` 都是**浅拷贝**（容器是新的，元素是共享的）。需要深拷贝时：

### 5.1 单层深拷贝（元素级别）

```objc
// NSArray
NSArray *deepArr = [[NSArray alloc] initWithArray:arr copyItems:YES];
// 对每个元素调用 copy，要求元素都遵循 NSCopying 协议

// NSDictionary
NSDictionary *deepDict = [[NSDictionary alloc] initWithDictionary:dict copyItems:YES];
```

⚠️ 这只是**一层深拷贝**：如果元素本身是容器，里面的子元素仍然共享。

### 5.2 完全深拷贝（递归）

```objc
// 方法一：NSKeyedArchiver（序列化/反序列化）
NSArray *fullDeepArr = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSArray class]
                                                         fromData:[NSKeyedArchiver archivedDataWithRootObject:arr
                                                                                       requiringSecureCoding:NO
                                                                                                       error:nil]
                                                            error:nil];

// 方法二：递归实现
- (id)deepCopy:(id)obj {
    if ([obj conformsToProtocol:@protocol(NSMutableCopying)]) {
        return [obj mutableCopy];
    }
    return [obj copy];
}
```

### 5.3 深拷贝方案对比

| 方案 | 深度 | 要求 | 性能 |
|------|------|------|------|
| `copy` / `mutableCopy` | 浅拷贝 | — | 最快 |
| `initWithArray:copyItems:YES` | 一层深拷贝 | 元素遵循 NSCopying | 较快 |
| `NSKeyedArchiver` | 完全深拷贝 | 元素遵循 NSSecureCoding | 较慢 |

---

## 六、Block 的 copy

Block 的内存管理比较特殊：

| Block 类型 | 内存区域 | 是否需要 copy |
|-----------|---------|-------------|
| 没有捕获自动变量 | 全局区（`_NSConcreteGlobalBlock`） | 不需要（copy 返回自身） |
| 捕获了自动变量 | 栈（`_NSConcreteStackBlock`） | **需要 copy 到堆** |
| 已经在堆上 | 堆（`_NSConcreteMallocBlock`） | 不需要（copy 返回自身） |

```objc
// MRC 下必须手动 copy
void (^block)(void) = [^{
    NSLog(@"%@", localVar);
} copy];  // 从栈拷贝到堆

// ARC 下系统自动 copy（赋值给 __strong 变量、作为返回值、GCD API 参数等）
void (^block)(void) = ^{
    NSLog(@"%@", localVar);
};  // ARC 自动 copy 到堆
```

所以 `@property (nonatomic, copy) void(^block)(void)` 的 `copy` 是为了兼容 MRC 时代的写法，ARC 下 `strong` 也可以，但习惯上仍用 `copy`。

---

## 七、总结速查表

| 概念 | 要点 |
|------|------|
| **copy** | 返回不可变副本 |
| **mutableCopy** | 返回可变副本，一定是新对象 |
| **浅拷贝** | 只复制容器，元素共享 |
| **深拷贝** | 连元素也复制 |
| **不可变 copy 不可变** | 不产生新对象（返回自身） |
| **可变 copy / mutableCopy** | 一定产生新对象 |
| **容器 copy/mutableCopy** | 都是浅拷贝 |
| **NSString 用 copy** | 防止外部 NSMutableString 意外修改属性值 |
| **NSArray/NSDictionary 用 copy** | 同理，防止外部可变容器意外修改 |
| **真正深拷贝** | `initWithArray:copyItems:YES`（一层）或 `NSKeyedArchiver`（完全） |
| **Block copy** | 栈 Block 需要 copy 到堆，ARC 自动处理 |
