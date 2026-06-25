---
type: review
---
# iOS 数据持久化

> **iOS 有哪些数据持久化方案？分别适合存什么类型的数据？CoreData 和 SQLite 是什么关系？UserDefaults 底层是什么？**

## 一、iOS 六大数据持久化方案

### 1. UserDefaults

- **本质**：`plist` 文件存储（`~/Library/Preferences/com.xxx.xxx.plist`）
- **底层实现**：底层是一个**内存中的字典**，写入时通过 `cfpreferences` 框架异步序列化为 XML plist 文件；读取时会**缓存在内存**中，首次访问会全量加载
- **适合存什么**：少量键值对——用户设置、开关状态、首次启动标记等
- **不适合**：大对象（>几KB）、敏感数据（明文存储）、复杂对象图

```swift
let defaults = UserDefaults.standard
defaults.set("dark", forKey: "theme")
defaults.synchronize() // 通常不需要手动调用，系统会自动批量写入
```

> **⚠️ 面试追问**：`UserDefaults` 是线程安全的吗？—— 是的，`UserDefaults` 内部有锁保护，但 `synchronize()` 在 iOS 12+ 已被标记为 unnecessary，系统会在适当的时机（runloop 结束、应用退后台）自动写入磁盘。

---

### 2. Keychain（钥匙串）

- **本质**：iOS 系统级加密存储，数据存储在 `/var/Keychains/` 的 SQLite 数据库中，由 Security.framework 管理
- **适合存什么**：敏感数据——密码、token、证书、信用卡号等
- **特点**：AES-256 加密、跨应用共享（通过 Access Group）、应用卸载后数据保留

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "userToken",
    kSecValueData as String: "secret".data(using: .utf8)!
]
SecItemAdd(query as CFDictionary, nil)
```

---

### 3. Plist 文件 / JSON 文件

- **本质**：把对象序列化为 plist 或 JSON 格式写入文件系统
- **适合存什么**：配置文件、小规模静态数据、App 的 bundle 内资源
- **限制**：只支持 `NSString, NSNumber, NSArray, NSDictionary, NSDate, NSData` 及其嵌套组合

```swift
// 写入
let dict = ["name": "Tom", "age": 18] as [String: Any]
let url = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
    .appendingPathComponent("user.plist")
(dict as NSDictionary).write(to: url, atomically: true)

// 读取
if let data = NSDictionary(contentsOf: url) as? [String: Any] {
    print(data["name"])
}
```

---

### 4. SQLite

- **本质**：轻量级关系型数据库引擎（C 语言实现），直接操作 `.db` 文件
- **适合存什么**：结构化数据、需要复杂查询（JOIN/索引/聚合）的场景
- **iOS 中的使用方式**：
  - 直接使用 C API（`sqlite3_open`, `sqlite3_exec` 等）
  - 使用封装库：**FMDB**（Objective-C）、**SQLite.swift**（Swift）

```sql
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER);
SELECT * FROM users WHERE age > 18 ORDER BY name;
```

**特点**：
- 无服务器、零配置、单文件存储
- 支持事务、索引、触发器
- 适合中大规模数据（万级~百万级记录）

---

### 5. CoreData

- **本质**：Apple 官方的**对象图管理框架**（Object-Graph Manager），底层默认存储引擎是 SQLite（也支持 In-Memory 和 XML）
- **适合存什么**：复杂对象模型、对象间有复杂关系、需要 undo/redo、需要数据迁移的场景
- **核心概念**：

| 概念 | 说明 |
|------|------|
| **NSManagedObject** | 持久化对象，对应表中的一行 |
| **NSManagedObjectContext** | 内存中的"草稿区"，管理对象的增删改查 |
| **NSPersistentStoreCoordinator** | 负责和底层存储（SQLite/内存）通信 |
| **NSManagedObjectModel** | 数据模型定义（`.xcdatamodeld`） |

```swift
// 创建一条记录
let context = persistentContainer.viewContext
let user = User(context: context) // User 是 NSManagedObject 子类
user.name = "Tom"
user.age = 25
try? context.save()

// 查询
let request: NSFetchRequest<User> = User.fetchRequest()
request.predicate = NSPredicate(format: "age > %d", 18)
let results = try? context.fetch(request)
```

---

### 6. FMDB / 第三方数据库框架

- **FMDB**：SQLite 的 OC 封装，线程安全（`FMDatabaseQueue`），Swift 项目中也很常用
- **WCDB**（腾讯）：基于 SQLite，支持 ORM、损坏修复、加密
- **Realm**：独立于 SQLite 的移动端数据库，有自己的存储引擎，API 极简

---

## 二、CoreData 和 SQLite 是什么关系？

**一句话回答**：CoreData 是上层的对象图管理框架，SQLite 是它默认的底层存储引擎之一。

```
┌──────────────────────────────────────┐
│         你的业务代码（Swift/OC）       │
├──────────────────────────────────────┤
│     CoreData（对象图 + 持久化框架）    │  ← 管理对象生命周期、关系、验证、迁移
├──────────┬───────────┬───────────────┤
│  SQLite  │  In-Memory│   XML Store   │  ← 底层存储引擎
└──────────┴───────────┴───────────────┘
```

| 对比维度 | CoreData | SQLite |
|---------|----------|--------|
| **本质** | 对象图管理框架 | 关系型数据库引擎 |
| **数据模型** | 面向对象（Entity → Object） | 关系型（Table → Row） |
| **查询语言** | NSPredicate + KVC | SQL |
| **线程模型** | Context 不可跨线程，需多 Context | 需自行管理线程安全 |
| **数据迁移** | 轻量/自动迁移 | 手动写迁移 SQL |
| **undo/redo** | 原生支持 | 不支持 |
| **多表 JOIN** | 通过 Relationship 属性自动处理 | 手写 JOIN 语句 |
| **性能** | 有额外开销（对象图管理、KVO） | 更接近裸速 |
| **适用场景** | 复杂对象关系、Apple 生态 | 纯数据存储、跨平台 |

> **⚠️ 面试追问**：那实际项目中该怎么选？
> - 如果项目是纯 Swift、对象模型复杂、且需要和 Apple 生态（SwiftUI `@FetchRequest`、CloudKit 同步）深度集成 → **CoreData**
> - 如果追求性能、跨平台、或者团队更熟悉 SQL → **SQLite + FMDB/WCDB**
> - 如果数据量小、结构简单 → **UserDefaults / Plist**

---

## 三、各方案适用场景速查表

| 方案 | 适合存 | 容量 | 加密 | 跨应用 | 随应用卸载删除 |
|------|--------|------|------|--------|--------------|
| **UserDefaults** | 用户偏好、开关 | 小（KB级） | ❌ | ❌ | ✅ |
| **Keychain** | 密码、token、证书 | 小（KB级） | ✅ 系统级 | ✅ Access Group | ❌ 保留 |
| **Plist/JSON** | 配置、静态数据 | 小~中 | ❌ | ❌ | ✅ |
| **SQLite** | 结构化业务数据 | 大（GB级） | 可选 | ❌ | ✅ |
| **CoreData** | 复杂对象图 | 大（GB级） | 可选 | ❌ | ✅ |
| **Realm** | 业务数据、实时同步 | 大 | 可选 | ❌ | ✅ |
| **文件系统** | 图片、音视频、文档 | 不限 | 可选 | ❌ | ✅ |

---

## 四、补充：文件系统存储

除了以上方案，iOS 还可以直接使用文件系统：

| 目录 | 用途 | 会被系统清理？ | 会被 iCloud 备份？ |
|------|------|--------------|-------------------|
| **Documents/** | 用户生成的重要数据 | ❌ | ✅ |
| **Library/Caches/** | 可重新生成的缓存数据 | ✅（磁盘不足时） | ❌ |
| **Library/Preferences/** | UserDefaults plist | ❌ | ✅ |
| **tmp/** | 临时文件 | ✅（系统随时清理） | ❌ |

```swift
let documents = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
```

---

## 五、面试回答模板

> **Q：iOS 有哪些数据持久化方案？**
>
> iOS 主要有 **6 种**持久化方案，按复杂度递增：
>
> 1. **UserDefaults** — 底层是 plist 文件，适合存少量键值对（用户偏好、开关），系统会在 runloop 结束时自动写入磁盘，是线程安全的
> 2. **Keychain** — 系统级加密存储，适合存敏感数据（密码、token），应用卸载后数据保留
> 3. **Plist / JSON** — 序列化为文件，适合存配置和小规模静态数据
> 4. **SQLite** — 轻量关系型数据库，适合结构化数据和复杂查询，项目中常用 FMDB 封装
> 5. **CoreData** — Apple 官方的对象图管理框架，底层默认用 SQLite 存储，适合复杂对象模型、需要数据迁移和 undo/redo 的场景
> 6. **文件系统** — 直接读写文件，适合存图片、音视频等大二进制数据
>
> 关于 CoreData 和 SQLite 的关系：CoreData 是上层框架，管理对象生命周期、关系、验证、迁移；SQLite 是它默认的底层存储引擎。CoreData 本质上不等于数据库，它是**对象图持久化框架**。选择时看项目需求——需要 Apple 生态集成和对象关系管理用 CoreData，追求性能和跨平台用 SQLite。
