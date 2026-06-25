---
type: review
---

# isa 指针结构（arm64）

## 一、什么是 isa 指针

在 Objective-C 中，每个对象的第一个成员变量就是一个 `isa` 指针。它指向对象所属的**类对象（Class）**，从而让对象能够访问类的方法列表、协议、属性等信息。

```objc
// objc_object 结构（简化）
struct objc_object {
    isa_t isa;
};
```

isa 的作用：
- **对象 → 类**：通过 isa 找到类对象，获取方法列表
- **类 → 元类**：类对象的 isa 指向元类（Meta-Class），获取类方法
- **继承链**：通过 superclass 指针找到父类

---

## 二、为什么需要 Non-Pointer isa

在 arm64 之前，isa 就是一个普通的指针，直接指向 Class 地址。但这样做有一个问题：**浪费了 64 位中的大部分空间**。

arm64 下指针是 64 位，但实际使用的地址远少于 64 位（通常是 33~42 位）。Apple 利用剩余的位存储了**对象的引用计数、是否已析构、是否使用 Tagged Pointer 等信息**，这就是 **Non-Pointer isa**。

好处：
- **减少内存访问**：引用计数等信息直接存在 isa 中，不需要额外访问散列表
- **提升性能**：retain/release 操作更快
- **节省内存**：不需要额外的引用计数表空间

---

## 三、isa_t 联合体结构

arm64 下 isa 是一个**联合体（union）**，既可以当作指针使用，也可以当作位域（bitfield）使用：

```cpp
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

    // 位域结构（仅示意，不同版本有差异）
    struct {
        uintptr_t non_pointer        : 1;  // 0: 普通指针  1: non-pointer isa
        uintptr_t has_assoc          : 1;  // 是否关联对象
        uintptr_t has_cxx_dtor       : 1;  // 是否有 C++ 析构函数
        uintptr_t shiftcls           : 33; // 类指针（实际地址右移后的值）
        uintptr_t magic              : 6;  // 调试用，固定值 0x1a
        uintptr_t weakly_referenced  : 1;  // 是否被 weak 引用过
        uintptr_t deallocating       : 1;  // 是否正在 dealloc
        uintptr_t has_sidetable_rc   : 1;  // 引用计数是否溢出到 SideTable
        uintptr_t extra_rc           : 19; // 额外引用计数（实际值 = extra_rc + 1）
    };
};
```

---

## 四、各 bit 字段详解

| 位（bit） | 字段 | 含义 |
|-----------|------|------|
| 0 | `non_pointer` | 1 = Non-Pointer isa，0 = 普通指针 |
| 1 | `has_assoc` | 是否设置过关联对象（dealloc 时需要清理） |
| 2 | `has_cxx_dtor` | 是否有 C++ / ARC 析构函数（dealloc 时需要调用） |
| 3~35 | `shiftcls` | **类指针**，存储 Class 地址右移 3 位后的值 |
| 36~41 | `magic` | 调试用魔数，固定为 `0x1a`，用于验证 isa 有效性 |
| 42 | `weakly_referenced` | 是否被 weak 指针引用过（dealloc 时需要清理 weak 表） |
| 43 | `deallocating` | 对象是否正在执行 dealloc |
| 44 | `has_sidetable_rc` | 引用计数是否超过 19 位上限，溢出到 SideTable |
| 45~63 | `extra_rc` | 额外引用计数，实际引用计数 = extra_rc + 1 |

> **注意**：不同 iOS 版本的位域布局可能有细微差异，以上为 arm64 下的典型结构。

---

## 五、类指针的存储方式

类指针并不是直接存储的，而是**右移 3 位**后存储在 `shiftcls` 字段中：

```cpp
// 存储：右移 3 位
shiftcls = (uintptr_t)cls >> 3;

// 读取：左移 3 位还原
Class cls = (Class)(shiftcls << 3);
```

为什么右移 3 位？
- arm64 下对象地址**最低 3 位一定是 0**（8 字节对齐）
- 右移 3 位可以多存 3 位信息
- 实际类指针占 33 位，可寻址范围 = 2^36 = **64 GB**，足够覆盖进程地址空间

---

## 六、引用计数如何存储

Non-Pointer isa 把引用计数拆成两部分：

1. **`extra_rc`（19 位）**：直接存在 isa 中，实际值 = extra_rc + 1
2. **`has_sidetable_rc`（1 位）**：标记是否溢出到 SideTable
3. **SideTable**：当引用计数超过 2^19（约 52 万）时，溢出到全局 SideTable 的 refcnts 哈希表

```
引用计数 = 1 + extra_rc + (has_sidetable_rc ? SideTable中的值 : 0)
```

retain 流程（简化）：
```
if (extra_rc < 2^19 - 1) {
    extra_rc++;           // 直接 isa + 1，极快
} else {
    // 溢出到 SideTable
    has_sidetable_rc = 1;
    SideTable[id].refcnts += (1 << 1);
}
```

---

## 七、代码验证

### 1. 验证 Non-Pointer isa

```objc
// 打印 isa 的 bits 值
NSObject *obj = [NSObject new];
NSLog(@"isa bits: %p", *((void **)(__bridge void *)obj));

// 如果最低位是 1，说明是 non-pointer isa
uintptr_t isaValue = *((uintptr_t *)obj);
NSLog(@"non-pointer: %d", isaValue & 1);
```

### 2. 验证类指针提取

```objc
// 从 isa 中提取类指针
uintptr_t isaValue = *((uintptr_t *)obj);
uintptr_t shiftcls = (isaValue >> 3) & 0x7FFFFFFFF;  // 取 33 位
Class cls = (__bridge Class)(void *)(shiftcls << 3);
NSLog(@"class from isa: %@", NSStringFromClass(cls));

// 验证与 object_getClass 一致
NSLog(@"object_getClass: %@", NSStringFromClass(object_getClass(obj)));
```

### 3. 验证引用计数

```objc
// 注意：ARC 下不能直接 retain，这里用 CF 层演示
NSObject *obj = [NSObject new];

// 打印 extra_rc
uintptr_t isaValue = *((uintptr_t *)obj);
uintptr_t extra_rc = (isaValue >> 45) & 0x7FFFF;  // 取 19 位
NSLog(@"extra_rc (should be 0): %lu", extra_rc);

// retain 后
CFRetain((__bridge CFTypeRef)obj);
isaValue = *((uintptr_t *)obj);
extra_rc = (isaValue >> 45) & 0x7FFFF;
NSLog(@"extra_rc after retain (should be 1): %lu", extra_rc);

CFRelease((__bridge CFTypeRef)obj);
```

---

## 八、isa 的指向关系

```
实例对象 (instance)
    │ isa
    ▼
类对象 (class) ──────────────────┐
    │ isa                        │
    ▼                            │ superclass
元类 (meta-class)                │
    │ isa                        ▼
    ▼                        父类对象 (superclass)
根元类 (root meta-class)          │ isa
    │ isa                        ▼
    ▼                        父元类 (super meta-class)
根元类自己（指向自己）              │ isa
                                 ▼
                             根元类 (root meta-class)
```

- 实例的 isa → 类对象
- 类对象的 isa → 元类
- 元类的 isa → 根元类
- 根元类的 isa → 根元类自己
- 所有元类的 superclass 链最终指向根元类，根元类的 superclass 是 NSObject 类对象

---

## 九、面试回答模板

**Q：请介绍一下 arm64 下 isa 的结构？**

> arm64 下 isa 不再是一个简单的指针，而是一个叫 `isa_t` 的联合体。它既能当指针用，也能当位域用。64 位中：
>
> - 第 0 位标记是否是 non-pointer isa
> - 第 1~2 位记录关联对象和 C++ 析构信息
> - 第 3~35 位存储类指针（右移 3 位后的值）
> - 第 36~41 位是调试用的魔数
> - 第 42~44 位记录 weak 引用、dealloc 状态、引用计数是否溢出
> - 第 45~63 位存储额外引用计数（19 位，最多约 52 万）
>
> 这样做的好处是：retain/release 时不需要额外查表，直接操作 isa 的位域就行，性能很高。只有引用计数溢出时才会存到全局的 SideTable 中。
>
> 提取类指针时，需要把 shiftcls 字段左移 3 位还原成真正的地址，因为 arm64 下对象地址都是 8 字节对齐的，最低 3 位一定是 0。

---

## 十、与其他笔记的关联

- [[Runtime]] — isa 是 Runtime 消息机制的基础
- [[iOS/tagged-pointer]] — 另一种指针优化技术，小对象直接把值编码进指针
- [[内存管理]] — 引用计数通过 isa 位域存储
- [[Block 闭包]] — Block 也是对象，也有 isa 指针
