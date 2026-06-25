---
type: review
---
# iOS 应用的完整启动流程

> **从用户点击 App 图标开始，到用户看到第一屏画面，中间发生了什么？越详细越好。**

## 阶段一：点击图标 → 进程创建

1. **Springboard 接收触摸事件**：用户点击 App 图标，Springboard（iOS 的桌面管理进程）捕获这个触摸事件。
2. **发起 Launch 请求**：Springboard 通过 `launchd` 服务向系统内核发起进程创建请求，调用 `posix_spawn` 或 `fork + exec`。
3. **内核创建进程**：Mach 内核分配进程结构（`task`、`thread`），分配虚拟地址空间，创建主线程。
4. **dyld 加载**：内核将控制权交给 **dyld**（dynamic linker），dyld 是用户态第一个运行的代码。

## 阶段二：dyld 加载过程

1. **加载 dylib**：dyld 按照依赖图依次加载：
   - `libSystem`（系统核心库）
   - `UIKit`、`Foundation`、`CoreFoundation` 等系统框架
   - App 自身依赖的其他 dylib
2. **Rebase & Bind**：
   - **Rebase**：修正内部指针（因为 ASLR 导致每次加载地址不同）
   - **Bind**：解析外部符号引用，将外部函数/变量指向实际地址
3. **ObjC Runtime 初始化**：
   - 注册所有 ObjC 类
   - 处理 Category
   - 执行 `+load` 方法（按 父类 → 子类 → 分类 的顺序）
4. **执行 Initializer**：
   - C++ 静态构造函数
   - `__attribute__((constructor))` 函数
   - `+initialize` 方法（首次使用类时触发）

## 阶段三：main() 入口

1. **main() 函数执行**：dyld 完成后，调用 App 的 `main()` 函数。
2. **UIApplicationMain()**：在 `main()` 中通常调用 `UIApplicationMain()`，它会：
   - 创建 `UIApplication` 单例
   - 创建 `AppDelegate` 实例
   - 创建主 `UIWindow`
   - 启动主 RunLoop

## 阶段四：AppDelegate 回调

1. **application:didFinishLaunchingWithOptions:**：
   - 这是开发者最熟悉的入口
   - 初始化第三方 SDK、配置全局设置、设置根视图控制器等
   - ⚠️ 这里做的事情越多，启动越慢

***

## 阶段四补：UIScene 生命周期（iOS 13+）

> iOS 13 引入了 `UIScene` API，支持多窗口（iPad 多任务、macOS Catalyst），**App 的 UI 生命周期从 AppDelegate 拆分到了 SceneDelegate**。

### 架构变化

| iOS 12 及以前                             | iOS 13+                                            |
| -------------------------------------- | -------------------------------------------------- |
| AppDelegate 管理 Window 和 UI 生命周期        | AppDelegate 只管 App 级生命周期，Window 交给 SceneDelegate   |
| 一个 App = 一个 Window                     | 一个 App = 多个 Scene，每个 Scene 有自己的 Window             |
| `application:didFinishLaunching` 设置 UI | `application:didFinishLaunching` 只做全局配置，**不设置 UI** |

### 启动流程（iOS 13+ UIScene 场景）

1. **application:didFinishLaunchingWithOptions:**：
   - 只做全局初始化（第三方 SDK、推送注册等）
   - **不再创建 Window 和 rootViewController**
2. **application:configurationForConnectingSceneSession:**：
   - 系统询问 AppDelegate 每个 Scene 的配置
   - 返回 `UISceneConfiguration`，指定 SceneDelegate 类名和 Storyboard
3. **scene:willConnectToSession:options:**（SceneDelegate）：
   - **这是 iOS 13+ 中设置首屏 UI 的正确位置**
   - 创建 `UIWindowScene`
   - 创建 `UIWindow` 并设置 `window.rootViewController`
   - 调用 `window.makeKeyAndVisible()`
4. **sceneDidBecomeActive:**（SceneDelegate）：
   - Scene 进入前台活跃状态
   - 对应旧的 `applicationDidBecomeActive:`

### Info.plist 配置

```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>  <!-- iPhone 通常为 false -->
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
                <key>UISceneStoryboardFile</key>
                <string>Main</string>
            </dict>
        </array>
    </dict>
</dict>
```

### SceneDelegate 关键方法

```swift
// 1. Scene 创建时（首次启动 / 恢复）
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UISceneConnectionOptions)

// 2. Scene 进入前台
func sceneDidBecomeActive(_ scene: UIScene)

// 3. Scene 进入后台
func sceneDidEnterBackground(_ scene: UIScene)

// 4. Scene 即将进入前台（从后台恢复）
func sceneWillEnterForeground(_ scene: UIScene)

// 5. Scene 即将不活跃（如来电、通知中心下拉）
func sceneWillResignActive(_ scene: UIScene)
```

### 向后兼容

- 如果需要支持 iOS 12 及以下，保留 AppDelegate 中的 Window 创建逻辑
- iOS 13+ 设备会优先使用 SceneDelegate，忽略 AppDelegate 中的 Window 设置
- 可以用 `@available` 检查运行时版本

***

## 阶段五：首屏渲染

1. **Root ViewController 加载**：
   - 如果使用 Storyboard，系统根据 Info.plist / SceneConfiguration 自动实例化初始 ViewController
   - 如果纯代码，在 `scene:willConnectToSession:` 或 AppDelegate 中手动创建并设置 `window.rootViewController`
2. **viewDidLoad**：加载视图层级，解析 XIB/Storyboard 或执行代码布局
3. **viewWillAppear / viewDidAppear**：视图即将/已经出现
4. **layoutSubviews / drawRect**：执行自动布局计算，绘制内容
5. **首帧提交**：Core Animation 将视图树序列化为图层树（layer tree），提交给 Render Server（`BackBoard`）
6. **Render Server 合成**：Render Server 在独立进程中执行 GPU 合成，最终显示到屏幕
7. **用户看到首屏** 🎉

## 性能优化关键点

| 阶段                 | 优化方向                                                         |
| ------------------ | ------------------------------------------------------------ |
| dyld 加载            | 减少 dylib 数量（建议 < 6 个非系统库），使用 `@executable_path` 避免路径查找       |
| ObjC Runtime       | 减少类数量，避免在 `+load` 中做重活，用 `+initialize` 替代                    |
| pre-main 阶段        | 减少动态库数量（合并、懒加载）、减少 +load 方法（用 +initialize 替代）、减少 ObjC 类和分类数量 |
| main() 前           | 移除不需要的 `+load`，精简 `__attribute__((constructor))`             |
| didFinishLaunching | 延迟初始化非关键 SDK，避免同步 I/O；iOS 13+ 此处**不应设置 UI**                  |
| UIScene (iOS 13+)  | `scene:willConnectToSession:` 中尽早设置 Window，避免异步延迟            |
| 首屏渲染               | 用 LaunchScreen.storyboard 作为占位、减少首屏视图层级、避免首屏有网络请求            |
| 二进制重排              | 优化 page fault，把启动时需要的函数放在一起（`order_file`）                    |

## 两个时间指标

- **T1（pre-main）**：从进程创建到 `main()` 执行的时间，受 dylib 数量和 `+load` 影响
- **T2（首屏渲染）**：从 `main()` 到首帧显示的时间，受 `didFinishLaunching` 和视图加载影响

Apple 的 **App Launch Time** 目标：

- 冷启动 < 400ms（理想状态）
- 热启动 < 200ms
