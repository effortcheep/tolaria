---
type: Note
---
# iOS 架构模式

> **MVC、MVVM、VIPER 分别是什么？Apple 原生 MVC 有什么问题？MVVM 是怎么解决这些问题的？你实际项目中用过哪种？**

***

## 一、三种架构模式概览

### 1. MVC（Model-View-Controller）

Apple 官方推荐的架构，三者职责：

| 角色             | 职责                     |
| -------------- | ---------------------- |
| **Model**      | 数据模型、业务逻辑              |
| **View**       | UI 展示（UIView 及其子类）     |
| **Controller** | 协调 Model 和 View，处理用户交互 |

数据流：`View → Controller → Model → Controller → View`

### 2. MVVM（Model-View-ViewModel）

| 角色            | 职责                                  |
| ------------- | ----------------------------------- |
| **Model**     | 数据模型，与 MVC 中相同                      |
| **View**      | UIView + UIViewController（只负责绑定和展示） |
| **ViewModel** | 将 Model 转换为 View 可直接使用的数据，处理业务逻辑    |

数据流：`View ↔ ViewModel → Model`\
核心机制：**数据绑定**（Binding），View 自动响应 ViewModel 的变化。

### 3. VIPER

将职责进一步拆分为五个模块：

| 角色             | 职责                            |
| -------------- | ----------------------------- |
| **View**       | 纯 UI 层，只负责展示                  |
| **Interactor** | 业务逻辑（类似 MVVM 的 ViewModel）     |
| **Presenter**  | 从 Interactor 获取数据，格式化后交给 View |
| **Entity**     | 纯数据模型（比 Model 更细粒度）           |
| **Router**     | 页面跳转 / 模块间导航                  |

数据流：`View → Presenter → Interactor → Entity → Interactor → Presenter → View`，Router 独立处理导航。

***

## 二、Apple 原生 MVC 的问题

Apple 的 MVC 在实际开发中容易退化为 **"Massive View Controller"**：

### 问题 1：Controller 臃肿

Apple 的 MVC 中，Controller 同时持有 View 和 Model 的引用，天然成为"万能对象"。网络请求、数据解析、UI 更新、业务逻辑、页面跳转……全堆在 Controller 里，一个文件动辄上千行。

### 问题 2：View 和 Model 没有直接通信

Apple 设计中 View 和 Model 是隔离的，所有交互必须经过 Controller。这意味着 Controller 无法被拆分或复用，因为它同时承担了两份胶水代码。

### 问题 3：难以单元测试

Controller 与 UIKit 强耦合，测试需要依赖真实 UI 环境。业务逻辑嵌在 Controller 里，无法独立测试。

### 问题 4：复用性差

业务逻辑和 UI 逻辑混在一起，想复用某个功能模块，必须连带整个 Controller 一起搬。

***

## 三、MVVM 如何解决这些问题

### 解决方案 1：抽出 ViewModel，Controller 瘦身

```typescript
// MVC 中：所有逻辑在 ViewController
class ProfileVC: UIViewController {
    func loadUser() {
        API.fetchUser { [weak self] user in
            self?.nameLabel.text = user.name
            self?.avatarView.kf.setImage(with: user.avatar)
            self?.levelBadge.text = "Lv.\(user.level)"
        }
    }
}

// MVVM 中：ViewController 只做绑定
class ProfileVC: UIViewController {
    var viewModel: ProfileViewModel!

    func bind() {
        viewModel.name.bind { [weak self] in self?.nameLabel.text = $0 }
        viewModel.avatarURL.bind { [weak self] in self?.avatarView.kf.setImage(with: $0) }
        viewModel.levelText.bind { [weak self] in self?.levelBadge.text = $0 }
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        bind()
        viewModel.loadUser()
    }
}
```

### 解决方案 2：数据绑定，消除手动同步

MVVM 的核心是 **响应式绑定**，View 自动监听 ViewModel 的变化。iOS 中常用实现方式：

| 方式                          | 说明                                  |
| --------------------------- | ----------------------------------- |
| **Closure / Delegate**      | 轻量级，手动调用，无第三方依赖                     |
| **Combine**                 | Apple 原生响应式框架（iOS 13+）              |
| **RxSwift**                 | 社区最流行的响应式框架                         |
| **@Observable（Swift 5.9+）** | 最新宏，配合 `withObservationTracking` 使用 |

### 解决方案 3：可测试性

ViewModel 不依赖 UIKit，可以直接单元测试：

```swift
func testLevelText() {
    let vm = ProfileViewModel()
    vm.user = User(name: "Tom", level: 5)
    XCTAssertEqual(vm.levelText, "Lv.5")
}
```

### 解决方案 4：职责清晰，可复用

ViewModel 只关心数据转换和业务逻辑，不关心 UI 实现。同一个 ViewModel 可以被不同 View 使用（iPhone / iPad / watchOS）。

***

## 四、VIPER 的优缺点

### 优点

- **极致解耦**：每个模块职责单一，适合大型团队并行开发
- **可测试性最好**：每个组件都可以独立 mock 和测试
- **模块化**：每个 VIPER 模块是一个独立单元，天然适合组件化

### 缺点

- **样板代码多**：一个页面至少 5 个文件（View / Interactor / Presenter / Entity / Router），简单页面显得过度设计
- **学习曲线陡**：新手难以理解数据流向
- **维护成本高**：模块间通信协议（Protocols）众多，修改一处牵连多处

### 适用场景

- 大型项目（10+ 人团队）
- 需要长期维护、频繁迭代的 App
- 模块数量 20+ 的复杂应用

***

## 五、三种架构对比

| 维度                | MVC      | MVVM                  | VIPER   |
| ----------------- | -------- | --------------------- | ------- |
| **文件数量**          | 少（2-3 个） | 中（3 个）                | 多（5+ 个） |
| **学习成本**          | 低        | 中                     | 高       |
| **Controller 大小** | 大（易臃肿）   | 小                     | 最小      |
| **可测试性**          | 差        | 好                     | 最好      |
| **适合团队规模**        | 1-3 人    | 3-8 人                 | 8+ 人    |
| **适合项目规模**        | 小型       | 中型                    | 大型      |
| **Apple 支持**      | 原生支持     | Combine / @Observable | 无原生支持   |

***

## 六、实际项目经验

### 中小型项目：MVVM + Combine

```swift
// 典型的 MVVM + Combine 实现
class ArticleListViewModel {
    @Published var articles: [Article] = []
    @Published var isLoading = false
    @Published var error: Error?

    private var cancellables = Set<AnyCancellable>()

    func fetchArticles() {
        isLoading = true
        API.fetchArticles()
            .receive(on: DispatchQueue.main)
            .sink(receiveCompletion: { [weak self] completion in
                self?.isLoading = false
                if case .failure(let err) = completion {
                    self?.error = err
                }
            }, receiveValue: { [weak self] articles in
                self?.articles = articles
            })
            .store(in: &cancellables)
    }
}
```

**选择理由**：

- Swift 原生支持 Combine，无需引入第三方库
- ViewModel 可独立测试
- 代码量适中，不会过度设计

### 大型项目：VIPER 模块化

对于功能模块众多的大型 App，可以将每个功能模块拆为 VIPER 单元，通过 Router 管理模块间跳转，通过 Protocol 降低耦合。

**实际做法**：

- 每个功能模块是一个独立的 Swift Package 或 Framework
- 模块间通过 Router Protocol 通信
- 共享的 Entity 和 Service 放在公共层

***

## 七、面试回答模板

> **Q：你项目中用过哪种架构？**\
> 我在项目中主要使用 **MVVM**，配合 Combine 做数据绑定。选择 MVVM 的原因是：它在代码量和可维护性之间取得了好的平衡——比 MVC 更清晰，比 VIPER 更轻量。\
> 具体做法是：ViewController 只负责 UI 绑定和用户交互，ViewModel 持有业务逻辑和数据转换，Model 是纯数据层。通过 Combine 的 `@Published` 属性驱动 UI 更新，ViewController 订阅 ViewModel 的输出。\
> 之前也评估过 VIPER，但对于我们的团队规模（2 人）和项目规模（中型 App），VIPER 的样板代码太多，投入产出比不高。MVVM 足够应对当前的复杂度。
