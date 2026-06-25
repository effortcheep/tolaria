---
type: review
---

# APNs 推送原理

## 一、APNs 是什么

APNs（Apple Push Notification Service）是苹果提供的**远程推送通知服务**，是 iOS/macOS/watchOS 等平台唯一合法的远程推送通道。

核心作用：**在 App 不在前台运行时，也能向用户发送消息。**

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  你的服务器  │───▶│  APNs 服务器 │───▶│   苹果设备   │───▶│   App    │
│ Provider  │    │ (苹果中继)  │    │  (iPhone)  │    │ (弹窗/角标) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

**为什么需要 APNs？**
- App 不可能一直保持与服务器的长连接（耗电、耗流量）
- 苹果不允许第三方推送通道（iOS 没有后台长连接保活）
- APNs 是苹果设备与服务器之间的**唯一可信推送通道**

## 二、推送完整流程

### 2.1 注册流程（一次性）

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│   App   │────────▶│   iOS   │────────▶│  APNs   │
│registerForRemote │  系统层   │         │  服务器   │
│Notifications     │         │         │         │
└─────────┘         └─────────┘         └─────────┘
      │                                      │
      │◀──────── deviceToken ────────────────│
      │                                      │
      │──── deviceToken 发送给你的服务器 ──────▶│
```

```swift
// 1. App 向系统注册推送
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
        if granted {
            DispatchQueue.main.async {
                application.registerForRemoteNotifications()
            }
        }
    }
    return true
}

// 2. 系统向 APNs 请求 deviceToken，成功回调
func application(_ application: UIApplication,
                 didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    print("deviceToken: \(token)")
    // 3. 将 deviceToken 发送给你的后端服务器保存
    APIService.registerPushToken(token)
}

// 4. 注册失败回调
func application(_ application: UIApplication,
                 didFailToRegisterForRemoteNotificationsWithError error: Error) {
    print("注册失败: \(error.localizedDescription)")
}
```

### 2.2 推送流程（每次推送）

```
┌──────────┐                        ┌──────────┐
│  你的服务器  │                        │  APNs    │
│ Provider  │                        │  服务器    │
└─────┬────┘                        └────┬─────┘
      │                                   │
      │ ① 发送推送请求                      │
      │ (deviceToken + payload)           │
      │──────────────────────────────────▶│
      │                                   │
      │                    ② 查找设备连接     │
      │                    ③ 推送到设备       │
      │                                   │────────▶ iPhone
      │                                   │
      │ ④ 返回响应                          │
      │◀──────────────────────────────────│
      │   成功 / 失败 / Token过期            │
```

### 2.3 设备接收流程

```
iPhone 收到推送
    │
    ▼
iOS 系统层处理
    │
    ├── App 在前台 → 直接交给 App 处理（不弹通知）
    │
    ├── App 在后台/未启动 → 系统弹出通知横幅
    │       │
    │       ├── 用户点击通知 → 打开 App，传递通知数据
    │       └── 用户忽略 → 通知留在通知中心
    │
    └── 静默推送 → 系统唤醒 App 后台处理（不弹窗）
```

## 三、推送 Payload 格式

### 3.1 基本 Payload

```json
{
    "aps": {
        "alert": {
            "title": "新消息",
            "subtitle": "来自张三",
            "body": "你好，明天开会记得来"
        },
        "badge": 1,
        "sound": "default",
        "category": "MESSAGE_CATEGORY"
    },
    "customKey": "自定义数据",
    "orderId": "12345"
}
```

### 3.2 Payload 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `aps.alert.title` | String | 通知标题 |
| `aps.alert.subtitle` | String | 通知副标题（iOS 10+） |
| `aps.alert.body` | String | 通知正文 |
| `aps.badge` | Int | 角标数字，0 = 清除角标 |
| `aps.sound` | String | 提示音，`default` 或自定义音效文件名 |
| `aps.category` | String | 通知类别（用于定义交互按钮） |
| `aps.content-available` | Int | 1 = 静默推送（后台唤醒） |
| `aps.mutable-content` | Int | 1 = 通知扩展（可修改内容） |
| `aps.interruption-level` | String | iOS 15+：`passive`/`active`/`timeSensitive`/`critical` |
| `aps.relevance-score` | Float | iOS 15+：0~1，系统决定展示优先级 |

### 3.3 Payload 大小限制

| 推送类型 | 最大大小 |
|---------|---------|
| 普通推送（旧协议） | 2 KB |
| 普通推送（HTTP/2） | 4 KB |
| VoIP 推送 | 5 KB |
| 通知扩展（mutable-content） | 4 KB（扩展处理后最终 4 KB） |

> 超过限制的推送会被 APNs 直接拒绝，返回 `413 Payload Too Large`。

## 四、DeviceToken 本质

### 4.1 什么是 DeviceToken

```
DeviceToken = APNs 用来标识「某台设备上的某个 App」的唯一令牌
```

- 由 APNs 在设备注册时生成
- 每个 App 在每台设备上都有一个唯一的 deviceToken
- **不是设备 UDID**，同一设备不同 App 的 token 不同
- **会变化**：系统升级、App 重装、用户重置推送权限等都可能导致 token 变化

### 4.2 DeviceToken 特性

| 特性 | 说明 |
|------|------|
| 长度 | 32 字节（64 个十六进制字符） |
| 唯一性 | 设备 + App 组合唯一 |
| 稳定性 | 可能变化，需每次注册时更新 |
| 过期 | 长期不活跃的 token 会被 APNs 清理 |
| 隐私 | 不包含设备信息，不可逆推设备 |

### 4.3 Token 刷新策略

```swift
// 每次 App 启动都注册推送，获取最新 token
// APNs 不保证 token 不变，所以需要每次检查
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions ...) -> Bool {
    // 不管之前有没有 token，每次启动都重新注册
    application.registerForRemoteNotifications()
    return true
}
```

## 五、APNs 通信协议

### 5.1 新协议（HTTP/2 + TLS）

当前 APNs 使用 **HTTP/2** 协议，Provider 与 APNs 之间通过 **TLS 双向认证** 建立连接。

```
Provider                          APNs
    │                               │
    │──── TLS 握手（双向证书认证）────▶│
    │◀──── TLS 连接建立 ────────────│
    │                               │
    │──── POST /3/device/{token} ──▶│
    │◀──── 200 OK / 错误码 ─────────│
    │                               │
    │  (连接复用，多路复用)            │
```

### 5.2 请求格式

```http
POST /3/device/{deviceToken} HTTP/2
Host: api.push.apple.com
Authorization: Bearer {JWT Token}   // 或使用证书
apns-topic: com.example.myapp       // App Bundle ID
apns-priority: 10                   // 10=立即, 5=省电
apns-expiration: 1640000000         // 过期时间戳
apns-push-type: alert               // alert/background/voip/etc

{
    "aps": {
        "alert": "Hello!",
        "sound": "default"
    }
}
```

### 5.3 认证方式

| 方式 | 说明 | 推荐度 |
|------|------|--------|
| **Token-based (JWT)** | 使用 .p8 密钥生成 JWT Token | ⭐⭐⭐ 推荐 |
| Certificate-based | 使用 .p12/.pem 证书 | ⭐⭐ 旧方式 |

```swift
// Token-based 认证 (推荐)
// 生成 JWT Token
func generateAPNsToken() -> String {
    let header = "{\"alg\":\"ES256\",\"kid\":\"YOUR_KEY_ID\"}"
    let claims = "{\"iss\":\"YOUR_TEAM_ID\",\"iat\":\(Int(Date().timeIntervalSince1970))}"
    // 使用 ES256 算法签名（p8 私钥）
    // ...
    return jwtToken
}
```

### 5.4 APNs 服务器地址

| 环境 | 地址 | 端口 |
|------|------|------|
| 开发 | `api.sandbox.push.apple.com` | 443 |
| 生产 | `api.push.apple.com` | 443 |
| 旧地址（已废弃） | `gateway.push.apple.com` | 2195 |

## 六、推送优先级与过期

### 6.1 优先级

| 优先级 | 值 | 行为 | 适用场景 |
|--------|---|------|---------|
| **High** | 10 | 立即推送，唤醒设备 | 聊天消息、来电、紧急通知 |
| **Low** | 5 | 批量推送，省电模式 | 营销推广、非紧急更新 |

```swift
// 高优先级 —— 立即送达
var request = URLRequest(url: url)
request.setValue("10", forHTTPHeaderField: "apns-priority")

// 低优先级 —— 批量优化
request.setValue("5", forHTTPHeaderField: "apns-priority")
```

### 6.2 过期机制

```swift
// 推送在 1 小时内有效，过期后 APNs 不再尝试投递
let expiration = Int(Date().addingTimeInterval(3600).timeIntervalSince1970)
request.setValue("\(expiration)", forHTTPHeaderField: "apns-expiration")
```

> 如果设备长时间离线，APNs 只保留**最后一条**推送，等设备上线后投递。

## 七、静默推送（Background Push）

### 7.1 什么是静默推送

静默推送**不会弹出通知界面**，而是唤醒 App 在后台执行代码。

```json
{
    "aps": {
        "content-available": 1
    },
    "syncType": "incremental",
    "syncData": { "newMessages": 5 }
}
```

### 7.2 静默推送特性

| 特性 | 说明 |
|------|------|
| 弹窗 | **不弹窗**，用户无感知 |
| 唤醒 | 系统唤醒 App，给约 **30 秒**后台执行时间 |
| 频率限制 | 系统会限制频率，一天几次到十几次不等 |
| 电量优先 | 低电量时系统可能不投递 |
| 回调 | `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)` |

```swift
// 静默推送处理
func application(_ application: UIApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                 fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    
    if let syncType = userInfo["syncType"] as? String {
        switch syncType {
        case "incremental":
            // 增量同步数据
            DataSyncManager.syncIncremental { result in
                completionHandler(result)
            }
        case "full":
            // 全量同步
            DataSyncManager.syncFull { result in
                completionHandler(result)
            }
        default:
            completionHandler(.noData)
        }
    } else {
        completionHandler(.noData)
    }
}
```

### 7.3 静默推送 vs 普通推送

| 对比项 | 普通推送 | 静默推送 |
|--------|---------|---------|
| 用户可见 | ✅ 弹窗/横幅 | ❌ 无界面 |
| 唤醒 App | 点击后唤醒 | 立即后台唤醒 |
| 执行时间 | 无限制 | ~30 秒 |
| 电量影响 | 小 | 较大（系统限制频率） |
| 用途 | 展示消息 | 后台数据同步 |
| `content-available` | 0 或不设 | 必须为 1 |

## 八、通知扩展（Notification Service Extension）

### 8.1 用途

在推送到达设备后、展示给用户之前，**拦截并修改通知内容**。

典型场景：
- 端到端加密消息解密
- 下载图片/视频附件
- 修改通知内容格式

### 8.2 使用步骤

```swift
// 1. 推送 Payload 加上 mutable-content
{
    "aps": {
        "alert": "你有一条新消息",
        "mutable-content": 1
    },
    "encryptedData": "base64..."
}

// 2. NotificationService.swift（扩展代码）
class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.contentHandler = contentHandler
        bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)
        
        guard let bestAttemptContent = bestAttemptContent else { return }
        
        // 解密消息
        if let encryptedData = bestAttemptContent.userInfo["encryptedData"] as? String {
            let decrypted = CryptoManager.decrypt(encryptedData)
            bestAttemptContent.body = decrypted
        }
        
        // 下载附件图片
        if let imageURL = bestAttemptContent.userInfo["imageURL"] as? String {
            downloadAndAttach(imageURL: imageURL, to: bestAttemptContent)
        } else {
            contentHandler(bestAttemptContent)
        }
    }
    
    // 超时回调（30秒内未完成处理）
    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler,
           let bestAttemptContent = bestAttemptContent {
            contentHandler(bestAttemptContent)
        }
    }
}
```

## 九、Provider 服务器实现

### 9.1 使用 HTTP/2 直接推送

```swift
// Swift 服务端（Vapor 框架示例）
import Vapor
import NIOHTTP2
import NIOSSL

func sendPushNotification(deviceToken: String, payload: [String: Any]) throws {
    let url = URL(string: "https://api.push.apple.com/3/device/\(deviceToken)")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("bearer \(jwtToken)", forHTTPHeaderField: "authorization")
    request.setValue("com.example.app", forHTTPHeaderField: "apns-topic")
    request.setValue("10", forHTTPHeaderField: "apns-priority")
    request.setValue("alert", forHTTPHeaderField: "apns-push-type")
    request.httpBody = try JSONSerialization.data(withJSONObject: payload)
    
    URLSession.shared.dataTask(with: request) { data, response, error in
        if let httpResponse = response as? HTTPURLResponse {
            switch httpResponse.statusCode {
            case 200: print("推送成功")
            case 400: print("请求格式错误")
            case 403: print("证书/Token 认证失败")
            case 410: print("DeviceToken 已过期，应删除")
            case 413: print("Payload 过大")
            case 429: print("发送太频繁，需限流")
            default:  print("其他错误: \(httpResponse.statusCode)")
            }
        }
    }.resume()
}
```

### 9.2 使用第三方库（推荐）

```swift
// 使用 apnswift（Swift 服务端）
import APNSwift
import NIOCore

struct PushPayload: APNSwiftNotification {
    let aps: APSPayload
    
    struct APSPayload: Codable {
        let alert: Alert?
        let badge: Int?
        let sound: String?
        
        struct Alert: Codable {
            let title: String
            let body: String
        }
    }
}

// 发送推送
let apns = try APNSwiftConnection(
    authenticationMethod: .jwt(
        privateKey: try .loadFrom(filePath: "AuthKey.p8"),
        keyIdentifier: "YOUR_KEY_ID",
        teamIdentifier: "YOUR_TEAM_ID"
    ),
    topic: "com.example.app",
    environment: .production
)

let payload = PushPayload(aps: .init(
    alert: .init(title: "新消息", body: "你好！"),
    badge: 1,
    sound: "default"
))

try await apns.send(payload, deviceToken: deviceToken)
```

## 十、APNs 错误码

| 状态码 | 含义 | 处理方案 |
|--------|------|---------|
| 200 | 成功 | — |
| 400 | 请求格式错误 | 检查 Payload JSON |
| 403 | 认证失败 | 检查证书/JWT |
| 405 | 方法错误 | 必须用 POST |
| 410 | DeviceToken 过期 | **删除该 Token** |
| 413 | Payload 过大 | 精简 Payload |
| 429 | 频率超限 | 降低发送速率 |
| 500 | APNs 内部错误 | 重试 |
| 503 | 服务不可用 | 稍后重试 |

> **重要**：收到 410 时，必须从数据库中删除该 token，它已经无效了。

## 十一、iOS 15+ 新特性

### 11.1 通知中断级别

```json
{
    "aps": {
        "alert": "你的外卖到了",
        "interruption-level": "timeSensitive",
        "relevance-score": 0.9
    }
}
```

| 级别 | 行为 | 场景 |
|------|------|------|
| `passive` | 静默入通知中心 | 新闻推送 |
| `active` | 默认行为，弹横幅 | 普通消息 |
| `timeSensitive` | 突破专注模式 | 快递到达、日程提醒 |
| `critical` | 突破静音，强制响铃 | 紧急安全警报（需特殊权限） |

### 11.2 通知摘要

系统会将低优先级通知**聚合为摘要**，在指定时间展示，减少打扰。

## 十二、常见问题与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 收不到推送 | 未授权通知权限 | 检查 `requestAuthorization` |
| 开发环境正常，生产不行 | 用了开发证书 | 生产环境用生产证书/p8 |
| 推送延迟 | 低优先级 / 省电模式 | 设置 `apns-priority: 10` |
| DeviceToken 频繁变化 | 系统行为 | 每次启动都重新上报 token |
| 静默推送不触发 | 系统限流 / 低电量 | 不要过于频繁，保证电量 |
| 通知扩展超时 | 处理超过 30 秒 | 优化下载/解密逻辑 |

## 十三、推送安全最佳实践

```swift
// 1. 不要在推送 Payload 中放敏感信息
// ❌ 错误
{"aps": {"alert": "你的密码是 123456"}}

// ✅ 正确 —— 只放通知文案，敏感数据 App 内拉取
{"aps": {"alert": "你有一条新消息"}, "messageId": "msg_001"}

// 2. 使用 Token-based 认证（JWT），不要用证书
// 3. 服务器端验证推送来源
// 4. 静默推送的内容也要加密
// 5. 及时清理无效 Token（410 响应）
```

## 十四、面试回答模板

> **Q：请介绍 APNs 推送原理**
>
> APNs 是苹果提供的远程推送通知服务，是 iOS 唯一合法的推送通道。
>
> **注册阶段**：App 调用 `registerForRemoteNotifications`，系统向 APNs 请求一个 deviceToken，App 将 token 上报给我们的后端服务器。
>
> **推送阶段**：后端将推送内容和 deviceToken 发送给 APNs，APNs 通过长连接将通知投递到目标设备，iOS 系统收到后根据 App 状态决定是弹窗还是交给 App 处理。
>
> **协议方面**：当前 APNs 使用 HTTP/2 协议，支持 Token-based（JWT + p8 密钥）和证书两种认证方式，推荐用 JWT 方式，一个密钥可以给多个 App 用。
>
> **关键细节**：
> - deviceToken 可能变化，每次启动都要重新上报
> - Payload 限制 4KB，超了会被拒绝
> - 设备离线时 APNs 只保留最后一条推送
> - 静默推送可以后台唤醒 App，但系统会限制频率
> - 通知扩展可以在展示前拦截修改通知内容
>
> **实际经验**：我在项目中使用了 MVVM 架构封装推送模块，通过 `UNUserNotificationCenter` 管理通知授权和交互，配合静默推送实现后台数据同步，并用通知扩展处理端到端加密消息的解密。服务器端使用 JWT 认证，对无效 token（410 响应）及时清理。

## 关联笔记

- [[iOS 应用的完整启动流程]] — 推送点击冷启动的完整流程
- [[多线程与 GCD]] — 静默推送后台处理涉及多线程
- [[内存管理]] — 推送 Payload 的内存优化
- [[Runtime]] — 推送相关的运行时机制
