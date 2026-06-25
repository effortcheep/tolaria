---
type: review
related_to: "[[内存管理]]"
---

# OC 中 weak 弱引用的实现原理

> **weak 指针是如何实现自动置 nil 的？底层数据结构是什么？销毁流程是怎样的？**

## 一、核心结论

`weak` 引用的实现依赖 **SideTable（散列表）** 机制：

- 每个被 `weak` 引用的对象，在运行时有一张对应的 **weak 引用表**
- `weak` 指针本身并不存储在对象的内存中，而是存在全局的 **SideTable** 里
- 当对象被释放（引用计数归零）时，运行时会遍历该对象的所有 weak 指针，**逐一置为 `nil`**

---

## 二、底层数据结构

### 1. SideTable（散列表）

Runtime 维护一个全局的 `SideTable` 数组（本质是哈希桶），通过对象地址哈希定位到对应的表项：

```
SideTable[]
┌─────────────────────────────────────────┐
│  spinlock_t     slock;      // 自旋锁   │
│  RefcountMap     refcnts;   // 引用计数表│
│  weak_table_t   weak_table; // weak 表   │
└─────────────────────────────────────────┘
```

### 2. weak_table_t（弱引用表）

每个被 weak 引用的对象，在 `weak_table` 中有一个条目：

```
weak_table_t {
    weak_entry_t *weak_entries;   // 哈希桶数组
    size_t        num_entries;    // 条目数量
    ...
}
```

### 3. weak_entry_t（弱引用条目）

存储了指向某个对象的 **所有 weak 指针**：

```
weak_entry_t {
    id             referent;         // 被引用的对象地址
    union {
        struct {
            weak_referrer_t *referrers;      // weak 指针数组（小型内联 / 大型动态）
            uintptr_t        inline_referrers[4]; // 内联存储（≤4个）
        };
        ...
    };
}
```

**关键点：** 一个对象可以被多个 `weak` 指针指向，所有 weak 指针的地址都记录在 `weak_entry_t` 中。

---

## 三、weak 注册流程

当执行 `__weak id obj = someObject;` 时，调用链为：

```
objc_storeWeak(&obj, someObject)
  └── weak_register_no_lock(&weak_table, &obj, someObject)
        ├── 1. 通过对象地址在 weak_table 中查找/创建 weak_entry_t
        ├── 2. 将 &obj（weak 指针的地址）添加到 referrers 数组
        └── 3. 如果是新条目，设置 referent = someObject
```

**核心操作：** 把"谁在 weak 引用我"记录下来。

---

## 四、weak 销毁/置 nil 流程

当对象引用计数归零、即将释放时，调用链为：

```
objc_deallocInstance(obj)
  └── objc_destructInstance(obj)
        └── weak_clear_no_lock(&weak_table, obj)
              ├── 1. 通过对象地址在 weak_table 中找到 weak_entry_t
              ├── 2. 遍历 referrers 数组中的所有 weak 指针
              ├── 3. 对每个 weak 指针执行 *referrer = nil
              └── 4. 从 weak_table 中删除该条目
```

**关键点：** 对象在 `dealloc` 过程中，运行时会自动将所有指向它的 weak 指针置为 `nil`，这就是为什么 weak 指针不会变成悬垂指针。

---

## 五、完整流程图

```
赋值阶段：  __weak id w = obj;
           ──────────────────────►  weak_register_no_lock()
                                     将 &w 存入 obj 的 weak_entry_t

使用阶段：  if (w) { [w doSomething]; }
           ──────────────────────►  正常读取，weak 不影响对象生命周期

释放阶段：  obj 引用计数归零
           ──────────────────────►  weak_clear_no_lock()
                                     遍历所有 weak 指针 → 置 nil
                                     删除 weak_entry_t
                                     对象内存被回收
```

---

## 六、为什么 weak 不增加引用计数

`weak` 注册到 SideTable 时，不会调用 `retain`。它只做"登记"操作——告诉运行时"我在 weak 引用这个对象"。对象的引用计数完全由 `strong` 指针控制。

这也是为什么 `weak` 需要配合 ARC 使用——它依赖运行时在对象释放时主动遍历置 nil，这个机制在 MRC 下不存在。

---

## 七、weak 与 autorelease 的配合

一个常见场景：

```objc
__weak id weakSelf = obj;
NSLog(@"%@", weakSelf); // 可能为 nil
```

在当前 RunLoop 迭代中 `obj` 可能已被 autorelease 池持有，到下一次 Drain 时才真正释放。`weak` 指针在对象 **真正 dealloc** 时才会被置 nil，而不是引用计数减到某个阈值。

---

## 八、性能考量

| 方面 | 说明 |
|------|------|
| **注册开销** | 需要哈希查找 + 数组插入，比 strong 赋值慢 |
| **释放开销** | 需要遍历所有 weak 指针置 nil，对象 weak 引用越多越慢 |
| **访问开销** | 和 strong 一样直接读取指针，无额外开销 |
| **适用场景** | delegate、闭包捕获、避免循环引用 |

---

## 九、要点总结

1. **weak 通过 SideTable（全局散列表）实现**，记录"对象 → weak 指针列表"的映射
2. **注册时**：将 weak 指针地址存入对象对应的 `weak_entry_t`
3. **释放时**：遍历 `weak_entry_t` 中所有 weak 指针，逐一置 `nil`
4. **weak 不增加引用计数**，只做登记，不影响对象生命周期
5. **线程安全**：SideTable 使用自旋锁保护，并发访问安全
