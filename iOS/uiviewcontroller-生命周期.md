---
type: review
---
# UIViewController 生命周期

> **UIViewController 从创建到销毁，完整的方法调用顺序是什么？viewDidLoad、viewWillAppear、viewDidAppear 等分别在什么时候调用？loadView 做了什么？如果用纯代码创建 UIViewController，哪些方法需要重写？**

## 一、完整生命周期方法调用顺序

```text
1. init / initWithNibName:bundle:    ← 初始化
2. loadView                          ← 创建视图（不要手动调用）
3. viewDidLoad                       ← 视图加载完成（只调一次）
4. viewWillAppear:                   ← 视图即将出现（可能多次）
5. viewWillLayoutSubviews            ← 即将布局子视图
6. viewDidLayoutSubviews             ← 布局完成
7. viewDidAppear:                    ← 视图已经出现
8. ————— 用户交互阶段 —————
9. viewWillDisappear:                ← 视图即将消失
10. viewDidDisappear:                ← 视图已经消失
11. dealloc                          ← 销毁
```

### 完整流程图

```text
VC 创建（init）
    ↓
第一次访问 self.view  或者 push 的时候
    ↓
loadView          ← 创建视图层级
    ↓
viewDidLoad       ← 视图加载完成，做一次性初始化
    ↓
viewWillAppear:   ← 视图即将显示，刷新数据、开始动画
    ↓
viewWillLayoutSubviews  ← 自动布局即将计算
viewDidLayoutSubviews   ← 自动布局计算完成
    ↓
viewDidAppear:    ← 视图已显示，开始请求、开启动画
    ↓
（用户操作，页面可见）
    ↓
viewWillDisappear:  ← 视图即将消失，保存状态、停止动画
    ↓
viewDidDisappear:   ← 视图已消失，释放资源
    ↓
dealloc             ← 对象销毁
```

***

## 二、每个方法详解

### 2.1 init 阶段

```objc
// 指定初始化方法
- (instancetype)initWithNibName:(NSString *)nibNameOrNil
                         bundle:(NSBundle *)nibBundleOrNil {
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // 初始化数据、变量
        // ⚠️ 不要访问 self.view
    }
    return self;
}

// 或者用 init（内部会调 initWithNibName:bundle:）
- (instancetype)init {
    return [super initWithNibName:nil bundle:nil];
}
```

**注意**：init 阶段不要访问 `self.view`，否则会提前触发 `loadView`。

### 2.2 loadView

```objc
// 作用：创建 VC 管理的根视图
// ⚠️ 不要手动调用！
// ⚠️ 不要调 super！（除非你用了 NIB）

- (void)loadView {
    // 纯代码创建视图
    UIView *view = [[UIView alloc] initWithFrame:[UIScreen mainScreen].bounds];
    view.backgroundColor = [UIColor whiteColor];
    self.view = view;
    
    // 在这里创建子视图
    UILabel *label = [[UILabel alloc] init];
    label.text = @"Hello";
    [self.view addSubview:label];
}
```

**loadView 的触发条件**：

- 第一次访问 `self.view` 时自动调用
- 之前没有设置过 `self.view`

**loadView 的三种情况**：

| 情况               | loadView 的行为  |
| ---------------- | ------------- |
| 有 NIB/Storyboard | 自动从 NIB 加载视图  |
| 重写了 loadView     | 使用你创建的视图      |
| 都没有              | 创建一个空白 UIView |

**⚠️ 不要同时重写 loadView 和使用 Storyboard/NIB**，否则会冲突。

### 2.3 viewDidLoad

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // ✅ 适合做的：
    // - 创建 UI 控件
    // - 初始化数据
    // - 网络请求（首次加载）
    // - 添加通知监听
    
    // ⚠️ 此时 view 的 frame 可能不是最终值
    // ⚠️ 不要做依赖 frame 的布局
}
```

**特点**：

- 只调用一次（整个生命周期）
- view 已经加载到内存
- frame 可能还是 `CGRectZero`

### 2.4 viewWillAppear:

```objc
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    // ✅ 适合做的：
    // - 刷新数据（每次页面即将显示）
    // - 开始动画
    // - 刷新导航栏状态
    // - 注册键盘通知
}
```

**特点**：

- 每次视图即将显示都会调用（push/pop/tab切换）
- 适合做数据刷新

### 2.5 viewWillLayoutSubviews / viewDidLayoutSubviews

```objc
- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    // 即将开始布局
}

- (void)viewDidLayoutSubviews {
    [super viewDidLayoutSubviews];
    
    // ✅ 适合做的：
    // - 依赖 frame 的布局调整
    // - 圆角、阴影等依赖 bounds 的设置
    // - 此时 frame 已经是自动布局计算后的最终值
}
```

**特点**：

- 每次布局变化都会调用
- `viewDidLayoutSubviews` 时 frame 是最终值

### 2.6 viewDidAppear:

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    // ✅ 适合做的：
    // - 开始播放动画
    // - 开始定位
    // - 弹出键盘
    // - 页面曝光统计
}
```

### 2.7 viewWillDisappear:

```objc
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    // ✅ 适合做的：
    // - 保存状态/数据
    // - 停止动画
    // - 收起键盘
    // - 取消网络请求
    // - 停止定位
}
```

### 2.8 viewDidDisappear:

```objc
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    
    // ✅ 适合做的：
    // - 移除通知监听
    // - 释放资源
    // - 停止定时器
}
```

### 2.9 dealloc

```objc
- (void)dealloc {
    // ✅ 适合做的：
    // - 移除通知监听
    // - 移除 KVO
    // - 停止定时器
    // - 取消网络请求
    
    // ⚠️ 不要访问 self.view
    // ⚠️ 不要调 [super dealloc]（ARC 下不需要）
}
```

***

## 三、frame 变化时机

```text
init                      → frame = CGRectZero（或自定义值）
loadView                  → frame = 屏幕大小（通常）
viewDidLoad               → frame = loadView 设置的值
viewWillLayoutSubviews    → frame = 自动布局计算后的值
viewDidLayoutSubviews     → frame = 最终值 ✅
viewWillAppear:           → frame = 最终值
viewDidAppear:            → frame = 最终值
```

**关键结论**：

- 依赖 frame 的代码放在 `viewDidLayoutSubviews` 或之后
- `viewDidLoad` 中的 frame 可能不是最终值

***

## 四、多次触发 vs 只触发一次

| 方法                     | 触发次数   | 场景              |
| ---------------------- | ------ | --------------- |
| init                   | 1 次    | 创建时             |
| loadView               | 1 次    | 第一次访问 self.view |
| viewDidLoad            | 1 次    | 视图加载完成          |
| viewWillAppear:        | **多次** | 每次视图即将显示        |
| viewWillLayoutSubviews | **多次** | 每次布局变化          |
| viewDidLayoutSubviews  | **多次** | 每次布局变化          |
| viewDidAppear:         | **多次** | 每次视图已显示         |
| viewWillDisappear:     | **多次** | 每次视图即将消失        |
| viewDidDisappear:      | **多次** | 每次视图已消失         |
| dealloc                | 1 次    | 销毁时             |

***

## 五、纯代码创建 UIViewController 的最小实现

```objc
@implementation MyViewController

// 1. 初始化
- (instancetype)init {
    self = [super initWithNibName:nil bundle:nil];
    if (self) {
        // 初始化数据
    }
    return self;
}

// 2. 创建视图（可选，系统会创建默认的 UIView）
- (void)loadView {
    UIView *view = [[UIView alloc] initWithFrame:[UIScreen mainScreen].bounds];
    view.backgroundColor = [UIColor whiteColor];
    self.view = view;
}

// 3. 视图加载完成
- (void)viewDidLoad {
    [super viewDidLoad];
    // 创建 UI、初始化数据
}

// 4. 视图即将显示
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    // 刷新数据
}

// 5. 视图已消失
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    // 释放资源
}

// 6. 销毁
- (void)dealloc {
    // 移除通知、KVO、定时器
}

@end
```

**最少只需要**：`viewDidLoad`（创建 UI）和 `dealloc`（清理资源）

***

## 六、常见使用场景对照

| 需求           | 应该放在                                       |
| ------------ | ------------------------------------------ |
| 创建 UI 控件     | `viewDidLoad`                              |
| 初始化数据        | `viewDidLoad` / `init`                     |
| 刷新页面数据       | `viewWillAppear:`                          |
| 依赖 frame 的布局 | `viewDidLayoutSubviews`                    |
| 开始动画         | `viewDidAppear:`                           |
| 保存状态         | `viewWillDisappear:`                       |
| 移除通知/KVO     | `dealloc` / `viewDidDisappear:`            |
| 网络请求         | `viewDidLoad`（首次）或 `viewWillAppear:`（每次刷新） |
| 页面统计         | `viewDidAppear:`                           |
| 弹出键盘         | `viewDidAppear:`                           |
| 停止定时器        | `viewWillDisappear:` / `dealloc`           |

***

## 七、特殊情况

### 7.1 push / pop

**A push B（首次）：**

```text
A viewWillDisappear
B loadView              ← 首次触发，创建视图
B viewDidLoad           ← 首次触发，只调一次
B viewWillAppear
B viewWillLayoutSubviews
B viewDidLayoutSubviews
A viewDidDisappear
B viewDidAppear
```

**B pop 回 A：**

```text
B viewWillDisappear
A viewWillAppear
A viewWillLayoutSubviews
A viewDidLayoutSubviews
B viewDidDisappear
A viewDidAppear
```

> `viewDidLoad` 只在首次加载时调用。pop 回来时 A 的视图还在内存中，不会再调 `viewDidLoad`。

### 7.2 内存警告

```objc
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    
    // 系统会调用 viewDidUnload（iOS 6 之前）
    // iOS 6+ 不再调用 viewDidUnload
    
    // 手动释放不需要的资源
    if ([self isViewLoaded] && self.view.window == nil) {
        self.view = nil;  // 释放视图，下次访问会重新 loadView + viewDidLoad
    }
}
```

### 7.3 present / dismiss

**A present B（首次）：**

```text
A viewWillDisappear
B loadView              ← 首次触发，创建视图
B viewDidLoad           ← 首次触发，只调一次
B viewWillAppear
B viewWillLayoutSubviews
B viewDidLayoutSubviews
B viewDidAppear         ← B 先完全显示
A viewDidDisappear      ← A 后消失
```

> ⚠️ 和 push/pop 不同：present 时 B 的 `viewDidAppear` 在 A 的 `viewDidDisappear` **之前**调用。

**B dismiss 回 A：**

```text
B viewWillDisappear
A viewWillAppear         ← A 的 viewDidLoad 不会再调（视图还在内存）
A viewWillLayoutSubviews
A viewDidLayoutSubviews
A viewDidAppear          ← A 先完全显示
B viewDidDisappear       ← B 后消失
```

> ⚠️ dismiss 时 A 的 `viewDidAppear` 在 B 的 `viewDidDisappear` **之前**调用。`viewDidLoad` 只在首次调用。

***

## 八、总结速查表

| 概念                        | 要点                            |
| ------------------------- | ----------------------------- |
| **loadView**              | 创建视图，不要手动调用，不要调 super（纯代码时）   |
| **viewDidLoad**           | 只调一次，适合创建 UI 和初始化数据           |
| **viewWillAppear:**       | 多次调用，适合刷新数据                   |
| **viewDidLayoutSubviews** | 多次调用，frame 已确定，适合依赖 frame 的布局 |
| **viewDidAppear:**        | 多次调用，适合开始动画和请求                |
| **viewWillDisappear:**    | 多次调用，适合保存状态和停止动画              |
| **dealloc**               | 只调一次，清理通知/KVO/定时器             |
| **纯代码最少实现**               | `viewDidLoad` + `dealloc`     |
| **frame 确定时机**            | `viewDidLayoutSubviews` 之后    |
| **viewDidLoad 调用次数**      | 1 次（视图还在内存就不会再调）              |
