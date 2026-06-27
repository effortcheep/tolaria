---
type: review
related_to: "[[React Native 的架构原理]]"
---
# Expo 的热更新与回滚机制

## 一、核心概念

### 1.1 什么是 OTA 更新

OTA（Over-The-Air）更新是指**不经过应用商店审核**，直接将新的 JS Bundle 推送到用户设备上的更新方式。

**关键限制**：OTA 只能更新 **JS Bundle 和静态资源**（图片、字体等），**不能更新原生代码**。如果修改了原生依赖（如新增原生模块、修改 app.json 中的权限配置），必须重新构建并提交应用商店。

### 1.2 Expo 的更新体系

| 概念 | 说明 |
| --- | --- |
| **EAS Update** | Expo 的 OTA 更新服务（替代旧的 expo-updates 服务器） |
| **EAS Build** | Expo 的云构建服务，生成原生安装包 |
| **Channel** | 部署通道，用于区分不同环境（production、staging、development） |
| **Runtime Version** | 原生兼容版本，决定哪些 OTA 更新可以安装到哪些原生包上 |
| **Branch** | 更新分支，类似 Git 分支，控制哪些更新推送到哪个通道 |

### 1.3 Runtime Version（运行时版本）

Runtime Version 是 Expo 热更新中**最关键的概念之一**，它解决的问题是：**OTA 更新必须与原生代码兼容**。

```json
{
  "expo": {
    "runtimeVersion": {
      "policy": "nativeVersion"
    }
  }
}
```

**两种策略**：

| 策略 | 生成规则 | 适用场景 |
| --- | --- | --- |
| `nativeVersion` | 直接使用原生版本号（如 1.2.0） | 大多数项目 |
| `appVersion` | 使用 version 字段 | 原生版本号不想频繁改动时 |

**工作原理**：
- 每次 EAS Build 会记录当前的 runtimeVersion
- EAS Update 发布时也会携带 runtimeVersion
- 设备请求更新时，**只下载 runtimeVersion 匹配的更新**
- 这保证了新 JS Bundle 不会在旧原生包上运行（避免崩溃）

```
原生包 v1.0.0 (runtimeVersion: "1.0.0")
  ├── 可安装 OTA 更新 A (runtimeVersion: "1.0.0") ✅
  ├── 可安装 OTA 更新 B (runtimeVersion: "1.0.0") ✅
  └── 不可安装 OTA 更新 C (runtimeVersion: "2.0.0") ❌

原生包 v2.0.0 (runtimeVersion: "2.0.0")
  └── 可安装 OTA 更新 C (runtimeVersion: "2.0.0") ✅
```

---

## 二、EAS Update 的工作原理

### 2.1 更新的完整生命周期

```
1. 开发者执行 eas update
   ↓
2. Metro 打包 → 生成 JS Bundle + 资源清单
   ↓
3. 上传到 EAS Server → 生成 Manifest（JSON）
   ↓
4. 用户设备启动 App → 向 EAS Server 发请求
   携带：runtimeVersion、platform、设备信息
   ↓
5. Server 返回 Manifest（包含最新 JS Bundle 的 URL + 资源清单）
   ↓
6. 设备对比本地版本和 Manifest → 有更新则下载
   ↓
7. 下载完成 → 写入本地存储 → 下次启动加载新 Bundle
   ↓
8. 如果新版本崩溃 → 自动回滚到上一个稳定版本
```

### 2.2 Manifest 的数据结构

当 App 向 EAS Server 请求更新时，服务器返回一个 JSON 格式的 Manifest：

```json
{
  "id": "a1b2c3d4-...",
  "createdAt": "2025-06-27T10:00:00Z",
  "runtimeVersion": "1.0.0",
  "launchAsset": {
    "url": "https://xxx.expo.dev/bundles/ios-abc123.js",
    "contentType": "application/javascript"
  },
  "assets": [
    {
      "hash": "md5hash",
      "key": "icon.png",
      "url": "https://xxx.expo.dev/assets/icon.png",
      "contentType": "image/png"
    }
  ],
  "metadata": {},
  "extra": {
    "expo": {
      "branch": "production"
    }
  }
}
```

### 2.3 App 侧的更新检查时机

```javascript
// 自动检查（默认行为，App 启动时自动检查）—— 无需额外代码

// 手动检查
import * as Updates from 'expo-updates';

async function checkForUpdate() {
  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      const result = await Updates.fetchUpdateAsync();
      if (result.isNew) {
        await Updates.reloadAsync();
      }
    }
  } catch (e) {
    console.log('更新检查失败:', e);
  }
}
```

### 2.4 expo-updates 的关键 API

| API | 作用 | 返回值 |
| --- | --- | --- |
| `checkForUpdateAsync()` | 检查是否有新更新 | `{ isAvailable: boolean }` |
| `fetchUpdateAsync()` | 下载新更新 | `{ isNew: boolean }` |
| `reloadAsync()` | 立即重启 App 加载新 Bundle | void |
| `Updates.updateId` | 当前运行的更新 ID | string 或 null |
| `Updates.createdAt` | 当前更新的创建时间 | Date 或 null |
| `Updates.isEmergencyLaunch` | 是否因崩溃回滚后紧急启动 | boolean |

---

## 三、回滚机制

### 3.1 自动回滚（Crash Recovery）

Expo 的自动回滚是**最后一道防线**：

```
App 启动
  ↓
加载新 JS Bundle
  ↓
运行成功？ → 是 → 正常使用
  ↓ 否（崩溃）
标记该版本为"失败"
  ↓
回滚到上一个稳定版本的 Bundle
  ↓
正常启动（isEmergencyLaunch = true）
```

**实现原理**：
- `expo-updates` 在加载新 Bundle 前，会保留旧 Bundle 的本地副本
- 新 Bundle 加载后如果 App 在启动阶段崩溃（未成功渲染首屏），下次启动会自动回退
- 回滚的判定基于**启动成功标志**：App 成功渲染首屏后才会标记为"稳定"

### 3.2 手动回滚（通过 EAS CLI）

当发现线上更新有 Bug 但 App 没有崩溃时，需要手动回滚：

```bash
# 方式一：查看当前通道的更新历史
eas update:list --branch production

# 方式二：重新发布旧版本的代码
git checkout <旧版本的commit>
eas update --branch production --message "回滚到旧版本"

# 方式三：通过 Branch 切换
eas channel:edit production --branch stable-v1.0
```

### 3.3 回滚的局限性

| 场景 | 能否 OTA 回滚 | 说明 |
| --- | --- | --- |
| JS 代码 Bug | ✅ 可以 | 发布旧代码的 OTA 更新即可 |
| 样式/UI 问题 | ✅ 可以 | 同上 |
| 原生模块 Bug | ❌ 不行 | 必须重新构建 + 应用商店审核 |
| 新增原生依赖后回滚 JS | ❌ 不行 | 旧 JS 可能调用不存在的原生方法 |
| 数据库 Schema 变更 | ❌ 不行 | 回滚 JS 无法回滚数据 |

### 3.4 Rollback Policy（回滚策略配置）

在 app.json 中可以配置回滚行为：

```json
{
  "expo": {
    "updates": {
      "url": "https://u.expo.dev/your-project-id",
      "fallbackToCacheTimeout": 30000,
      "checkAutomatically": "ON_LOAD"
    }
  }
}
```

| 配置项 | 说明 |
| --- | --- |
| `checkAutomatically: "ON_LOAD"` | App 每次启动时检查更新（默认） |
| `checkAutomatically: "ON_ERROR"` | 只在崩溃恢复时检查更新 |
| `checkAutomatically: "NEVER"` | 从不自动检查，完全手动控制 |
| `fallbackToCacheTimeout` | 等待更新下载的超时时间（ms），超时则用缓存版本 |

---

## 四、灰度更新（Phased Rollout）

### 4.1 什么是灰度更新

灰度更新是指**将新版本逐步推送给部分用户**，观察稳定性后再全量发布，降低线上事故的影响范围。

```
全量用户
  ├── 5% 用户 → 新版本 A（观察 1 小时）
  │     ├── 崩溃率正常？ → 扩大到 20%
  │     │                    ├── 稳定？ → 扩大到 50% → 100%
  │     │                    └── 异常？ → 回滚，排查问题
  │     └── 崩溃率升高？ → 立即回滚
  └── 95% 用户 → 继续使用旧版本
```

### 4.2 Expo 中实现灰度的三种方式

#### 方式一：通过 Branch + Channel 映射（推荐）

Expo 的 Channel 和 Branch 是多对多关系，可以灵活控制不同用户群看到哪个版本。

```
Channel: production → Branch: production（主版本）
Channel: production-canary → Branch: canary（灰度版本）
```

**操作步骤**：

```bash
# 1. 构建原生包
eas build --platform all --profile production

# 2. 发布灰度更新到 canary 分支
eas update --branch canary --message "灰度测试 v1.1.0"

# 3. 将 production 通道指向 canary 分支（开始灰度）
eas channel:edit production --branch canary

# 4. 观察指标，确认无问题后恢复
eas channel:edit production --branch production

# 5. 发布正式版本到 production 分支
eas update --branch production --message "正式发布 v1.1.0"
```

#### 方式二：通过 expo-updates 自定义逻辑

在 App 内根据用户标识决定是否接受更新：

```javascript
import * as Updates from 'expo-updates';
import AsyncStorage from '@react-native-async-storage/async-storage';

function isInCanaryGroup(userId) {
  const hash = simpleHash(userId);
  return hash % 100 < 5; // 前 5% 的用户
}

async function smartUpdateCheck(userId) {
  const isCanary = isInCanaryGroup(userId);
  const lastUpdateChannel = await AsyncStorage.getItem('updateChannel');

  if (isCanary || lastUpdateChannel === 'canary') {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      const result = await Updates.fetchUpdateAsync();
      if (result.isNew) {
        await AsyncStorage.setItem('updateChannel', 'canary');
        await Updates.reloadAsync();
      }
    }
  }
}
```

#### 方式三：通过 Server-Side 控制（最灵活）

搭建一个中间层服务，根据用户属性决定返回哪个 Manifest：

```
用户设备 → 请求更新 → 你的 API Server
                          ├── 灰度用户 → 转发到 EAS Server 的 canary 分支
                          └── 普通用户 → 转发到 EAS Server 的 production 分支
```

### 4.3 灰度监控指标

| 指标 | 告警阈值 | 说明 |
| --- | --- | --- |
| 崩溃率 | > 1% | 新版本的 crash-free rate 低于 99% |
| JS 错误率 | > 0.5% | 未捕获的 JS 异常 |
| ANR 率 | > 0.5% | Android 无响应 |
| 启动时间 | > 2s | 冷启动时间劣化 |
| 用户反馈 | 关键词监控 | 应用商店/客服反馈 |

### 4.4 Expo 中的监控集成

```javascript
import * as Updates from 'expo-updates';
import * as Sentry from '@sentry/react-native';

function initMonitoring() {
  const updateInfo = {
    updateId: Updates.updateId,
    createdAt: Updates.createdAt,
    isEmergencyLaunch: Updates.isEmergencyLaunch,
    channel: Updates.channel,
    runtimeVersion: Updates.runtimeVersion,
  };
  
  Sentry.setContext('expo-update', updateInfo);
  reportToAnalytics('app-launch', updateInfo);
}

// 监听崩溃回滚
if (Updates.isEmergencyLaunch) {
  reportToAnalytics('emergency-rollback', {
    updateId: Updates.updateId,
    timestamp: new Date().toISOString(),
  });
}
```

---

## 五、关键机制深入

### 5.1 expo-updates 的本地存储结构

`expo-updates` 在设备本地维护了一个版本管理系统：

```
App 沙箱/
  ├── expo-updates/
  │   ├── db.json              ← 版本数据库（记录所有已下载的更新）
  │   ├── bundles/
  │   │   ├── bundle-abc123.js ← 当前活跃的 JS Bundle
  │   │   └── bundle-def456.js ← 上一个稳定版本（用于回滚）
  │   └── assets/
  │       ├── icon.png
  │       └── splash.png
  └── ...
```

**db.json 的核心字段**：

| 字段 | 说明 |
| --- | --- |
| `id` | 更新的唯一标识 |
| `scopeKey` | 项目级别的隔离 key |
| `runtimeVersion` | 原生兼容版本 |
| `launchAsset` | 指向 JS Bundle 的路径 |
| `status` | 状态：`new` / `downloading` / `downloaded` / `launching` / `launched` |
| `isRollback` | 是否为回滚更新 |

### 5.2 Bundle 加载的完整流程

```
App 进程启动
  ↓
expo-updates 初始化
  ↓
读取 db.json → 找到 status = "launched" 的记录
  ↓
检查该 Bundle 是否存在且完整（文件 hash 校验）
  ↓
  ├── 完整 → 加载该 Bundle → 启动成功 → 正常运行
  │
  └── 文件损坏/缺失 → 标记为失败 → 尝试加载上一个稳定版本
                                        ↓
                                  ├── 找到 → 加载旧 Bundle → isEmergencyLaunch = true
                                  └── 没有 → 回退到打包时内置的 Bundle
```

### 5.3 更新下载的差量更新（Diff Update）

EAS Update 支持**差量更新**，只下载变化的部分：

| 更新类型 | 说明 | 数据量 |
| --- | --- | --- |
| 全量更新 | 下载完整的 JS Bundle | 通常 1~5 MB |
| 差量更新 | 只下载两个版本之间的 diff | 通常 10~100 KB |

差量更新的实现依赖于 **bsdiff** 算法：
1. Server 保存历史版本的 Bundle
2. 新版本发布时，Server 计算与上一版本的 diff
3. 设备请求更新时，如果 Server 知道设备当前版本，返回 diff patch
4. 设备应用 patch 生成新 Bundle

```javascript
// 差量更新对开发者是透明的，无需额外代码
// 但可以通过配置控制：
{
  "expo": {
    "updates": {
      "codeSigningCertificate": "./certs/code-signing.pem",
      "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
    }
  }
}
```

### 5.4 更新的签名验证

为了防止中间人攻击篡改更新内容，Expo 支持**代码签名**：

```bash
# 1. 生成密钥对
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# 2. 在 app.json 中配置公钥（嵌入 App 内）
{
  "expo": {
    "updates": {
      "codeSigningCertificate": "./public.pem",
      "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
    }
  }
}

# 3. 发布时用私钥签名
eas update --branch production --private-key-path ./private.pem
```

**验证流程**：
```
设备收到 Manifest → 检查签名 → 用内置公钥验证
  ├── 验证通过 → 下载 Bundle
  └── 验证失败 → 拒绝更新，使用当前版本
```

---

## 六、完整的发布流程

### 6.1 推荐的发布流水线

```
开发 → Git 提交
  ↓
EAS Build (staging profile)
  ↓
测试环境验证
  ↓
eas update --branch staging
  ↓
QA 验证
  ↓
eas update --branch canary
  ↓
灰度监控（1~24 小时）
  ├── 稳定 → eas update --branch production（全量发布）
  └── 异常 → 回滚 canary 到旧版本，排查问题
```

### 6.2 多环境配置示例

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "channel": "development"
    },
    "staging": {
      "distribution": "internal",
      "channel": "staging"
    },
    "production": {
      "channel": "production"
    }
  },
  "update": {
    "channel": "production"
  }
}
```

```bash
# 开发阶段
eas update --branch development --message "开发测试"

# staging 测试
eas update --branch staging --message "v1.1.0-rc1"

# 灰度发布
eas update --branch canary --message "v1.1.0-canary"

# 正式发布
eas update --branch production --message "v1.1.0 正式版"
```

---

## 七、回答问题

### ① Expo 的热更新和回滚是怎么做到的？

**热更新原理**：
- EAS Update 将 JS Bundle 和静态资源上传到 CDN，返回一个 Manifest（JSON）
- App 启动时向 EAS Server 请求 Manifest，对比 runtimeVersion 和更新 ID
- 有新版本则下载 Bundle 到本地存储，下次启动时加载新 Bundle
- 整个过程不经过应用商店，用户无感知

**回滚机制**：
- **自动回滚**：expo-updates 保留旧 Bundle 副本，如果新 Bundle 导致启动崩溃，下次自动回退到旧版本
- **手动回滚**：通过 `eas channel:edit` 将生产通道指向已知稳定的分支，或重新发布旧代码的 OTA 更新
- **局限**：OTA 只能回滚 JS 层的变更，原生代码的 Bug 必须重新构建提交商店

### ② 如何做灰度更新？

Expo 中三种灰度方案：

| 方案 | 复杂度 | 灵活性 | 说明 |
| --- | --- | --- | --- |
| **Branch + Channel 映射** | 低 | 中 | 用 eas channel:edit 切换通道指向，适合简单的灰度 |
| **App 内自定义逻辑** | 中 | 高 | 根据用户 ID 哈希决定是否接受更新，粒度更细 |
| **Server-Side 控制** | 高 | 最高 | 搭建中间层服务，按用户属性动态返回不同分支的 Manifest |

**推荐做法**：
1. 先用 Branch + Channel 做小范围灰度（5%~10%）
2. 观察崩溃率、JS 错误率、启动时间等指标 1~24 小时
3. 指标正常则全量发布，异常则立即回滚
4. 大型项目可结合 Server-Side 控制做更精细的灰度策略

---

## 八、EAS Update vs CodePush

| 维度 | EAS Update | CodePush（Microsoft） |
| --- | --- | --- |
| **维护者** | Expo 官方 | Microsoft（已停止维护） |
| **RN 版本支持** | RN 0.71+ | RN 0.59 ~ 0.73 |
| **Expo 集成** | 原生支持，零配置 | 需要手动配置 |
| **差量更新** | ✅ 支持 | ✅ 支持 |
| **代码签名** | ✅ 支持 | ❌ 不支持 |
| **Branch/Channel** | ✅ 原生支持 | ✅ 部署键（Deployment Key） |
| **回滚机制** | 自动 + 手动 | 自动 + 手动 |
| **自托管** | ❌ 只能用 Expo 云服务 | ✅ 可自建 CodePush Server |
| **价格** | 免费额度 + 付费计划 | 免费（自建服务器） |
| **状态** | 活跃维护 | 已停止维护（2024） |

**结论**：如果是新项目，推荐使用 EAS Update。如果已有 CodePush 集成且运行稳定，可以继续使用，但需要关注迁移计划。

---

## 九、常见问题

### Q1: 修改了原生代码还能用 OTA 更新吗？

**不能**。OTA 只能更新 JS Bundle 和静态资源。以下情况必须重新构建：
- 新增/修改原生模块（Native Module）
- 修改 `app.json` 中的权限、图标、启动图等
- 升级 React Native 版本
- 修改 iOS Podfile 或 Android build.gradle

### Q2: 更新下载后什么时候生效？

**默认下次启动生效**。如果想立即生效：

```javascript
import * as Updates from 'expo-updates';

// 下载完成后立即重启 App 加载新版本
await Updates.reloadAsync();
```

注意：`reloadAsync()` 会重新执行整个 App 初始化流程，用户会看到短暂的白屏。

### Q3: 如何知道用户用的是哪个版本？

```javascript
import * as Updates from 'expo-updates';

console.log(Updates.updateId);      // 当前更新的唯一 ID
console.log(Updates.createdAt);     // 更新的创建时间
console.log(Updates.runtimeVersion); // 运行时版本
console.log(Updates.channel);       // 通道（production/staging/...）
```

建议在 App 启动时将这些信息上报到监控系统（Sentry、Firebase Analytics 等）。

### Q4: 多个更新同时存在，设备会下载哪个？

设备只会下载**当前 runtimeVersion 匹配的最新一个更新**。EAS Server 的逻辑是：
1. 收到请求，提取设备的 `runtimeVersion` 和 `platform`
2. 在对应的 Branch 中找到最新的匹配更新
3. 如果设备当前已有的 `updateId` 与最新一致，返回 `304 Not Modified`
4. 否则返回新的 Manifest
