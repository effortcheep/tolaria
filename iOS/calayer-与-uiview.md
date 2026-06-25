---
type: review
related_to: "[[iOS]]"
---
# CALayer 与 UIView

> **UIView 和 CALayer 是什么关系？为什么有了 UIView 还需要 CALayer？哪些属性是 Layer 独有的？隐式动画是什么？**

## 一、UIView 和 CALayer 是什么关系

**UIView 是 CALayer 的高层封装**，每个 UIView 内部都有一个 `layer` 属性（`@property (nonatomic, readonly, strong) CALayer *layer`）。

```text
UIView
├── layer (CALayer)        ← 负责内容绘制和动画
├── 手势识别
├── 事件响应（hitTest: / touchesBegan:）
├── Auto Layout
└── 响应者链
```

**职责分工：**

|          | UIView                | CALayer                        |
| -------- | --------------------- | ------------------------------ |
| **内容绘制** | ❌ 委托给 layer           | ✅ 管理 contents / drawRect       |
| **事件响应** | ✅ 继承 UIResponder      | ❌ 不处理事件                        |
| **动画**   | ❌ 委托给 layer           | ✅ 管理动画和图层树                     |
| **布局**   | ✅ Auto Layout / frame | ✅ frame / bounds / anchorPoint |
| **视图层级** | ✅ subviews            | ✅ sublayers                    |

***

## 二、为什么有了 UIView 还需要 CALayer

**关注点分离（单一职责原则）：**

1. **跨平台**：CALayer 属于 QuartzCore 框架，可以在 macOS 和 iOS 之间共享。UIView 继承自 UIResponder，是 iOS 独有的。
2. **职责单一**：UIView 专注事件处理和布局，CALayer 专注内容绘制和动画。一个 layer 可以被多个 view 共享（理论上），一个 view 可以管理多个 layer。
3. **性能**：Core Animation 是独立的渲染进程（Render Server），layer 树可以直接提交给 GPU 合成，不需要经过 UIView 的事件处理逻辑。

> 简单说：**UIView 管「交互」，CALayer 管「显示」**。

***

## 三、CALayer 独有的属性

这些属性 UIView 没有直接暴露，或者只是简单转发给 layer：

### 3.1 内容相关

```objc
// 寄宿图（bitmap content）
layer.contents = (__bridge id)image.CGImage;  // 直接设置位图
layer.contentsGravity = kCAGravityResizeAspect;  // 类似 contentMode
layer.contentsScale = [UIScreen mainScreen].scale;  // 像素密度
layer.contentsRect = CGRectMake(0, 0, 0.5, 0.5);  // 显示图片的哪个区域
layer.contentsCenter = CGRectMake(0.25, 0.25, 0.5, 0.5);  // 九宫格拉伸区域
```

### 3.2 边框和圆角

```objc
layer.cornerRadius = 10;           // 圆角（会裁剪子图层）
layer.borderWidth = 1;             // 边框宽度
layer.borderColor = [UIColor redColor].CGColor;  // 边框颜色
layer.masksToBounds = YES;         // 裁剪超出部分（对应 clipsToBounds）
```

### 3.3 阴影（⚠️ 和圆角冲突）

```objc
layer.shadowOpacity = 0.5;         // 阴影透明度（0~1，默认0，即不显示）
layer.shadowOffset = CGSizeMake(0, 3);  // 阴影偏移
layer.shadowRadius = 5;            // 阴影模糊半径
layer.shadowColor = [UIColor blackColor].CGColor;  // 阴影颜色
layer.shadowPath = [UIBezierPath bezierPathWithRect:layer.bounds].CGPath;  // 阴影路径（优化性能）
```

> ⚠️ **圆角 + 阴影冲突**：`masksToBounds = YES` 会裁剪阴影。解决方案：用两个 layer，底层画阴影，上层画圆角+裁剪。

### 3.4 变换（3D 变换）

```objc
// 2D 变换（UIView 也有）
layer.affineTransform = CGAffineTransformMakeRotation(M_PI_4);

// 3D 变换（UIView 没有）
layer.transform = CATransform3DMakeRotation(M_PI_4, 0, 0, 1);
layer.transform = CATransform3DMakeScale(1.5, 1.5, 1);
layer.transform = CATransform3DMakeTranslation(0, 0, 50);  // z 轴平移

// 透视效果（m34 控制近大远小）
CATransform3D t = CATransform3DIdentity;
t.m34 = -1.0 / 500.0;  // 值越小透视越强
layer.transform = CATransform3DRotate(t, M_PI_4, 0, 1, 0);
```

### 3.5 其他独有属性

```objc
layer.opacity = 0.8;               // 透明度（UIView 的 alpha 转发到这里）
layer.backgroundColor = [UIColor whiteColor].CGColor;  // 背景色
layer.hidden = NO;                 // 是否隐藏
layer.doubleSided = YES;           // 背面是否显示（3D 翻转时）
layer.shouldRasterize = YES;       // 光栅化（缓存为位图，优化性能）
layer.rasterizationScale = [UIScreen mainScreen].scale;  // 光栅化分辨率
```

***

## 四、隐式动画（Implicit Animation）

### 4.1 什么是隐式动画

**对 CALayer 属性的修改，默认会有动画效果**，不需要显式添加 `CABasicAnimation`。这就是隐式动画。

```objc
// 隐式动画 — 直接修改属性，自动产生 0.25s 的动画
[CATransaction begin];
[CATransaction setAnimationDuration:0.5];
layer.cornerRadius = 20;
layer.backgroundColor = [UIColor redColor].CGColor;
[CATransaction commit];
```

**哪些属性有隐式动画：**

- 几乎所有 CALayer 的可动画属性：`bounds`、`position`、`transform`、`opacity`、`backgroundColor`、`cornerRadius`、`borderWidth`、`shadowOffset` 等
- `contents` 没有隐式动画（图片切换是离散的）

### 4.2 为什么 UIView 的属性修改没有隐式动画

**UIView 默认禁用了 layer 的隐式动画。**

当你在 UIView 的动画块中修改属性时：

```objc
[UIView animateWithDuration:0.3 animations:^{
    self.view.frame = newFrame;
}];
```

UIView 内部会：

1. 开启一个 `CATransaction`
2. 禁用 layer 的隐式动画
3. 修改 layer 的对应属性
4. 创建显式的 `CABasicAnimation` 添加到 layer 上
5. 提交事务

所以你看到的动画是 UIView 显式添加的，不是隐式动画。

### 4.3 隐式动画的底层机制

```text
修改 CALayer 的属性
    ↓
CALayer 检查是否有 actionForKey: 的代理
    ↓
有代理？→ 调用 actionForLayer:forKey: 返回动画对象
    ↓
没有代理？→ 查找 actions 字典
    ↓
没有？→ 查找 style 字典
    ↓
都没有？→ 执行默认动画（0.25s）
```

**UIView 作为 layer 的代理，**`actionForLayer:forKey:` **返回** `[NSNull null]`**，表示禁用隐式动画。** 只有在 UIView 的动画块中才会返回正常的动画对象。

```objc
// 验证：在动画块外修改 layer 属性
NSLog(@"%@", [self.view actionForLayer:self.view.layer forKey:@"backgroundColor"]);
// 输出: <null>（禁用动画）

[UIView animateWithDuration:0.3 animations:^{
    NSLog(@"%@", [self.view actionForLayer:self.view.layer forKey:@"backgroundColor"]);
    // 输出: <CABasicAnimation: 0x...>（显式动画）
}];
```

### 4.4 在 UIView 动画块中直接操作 layer

```objc
// ✅ 在动画块中修改 layer 属性，会产生动画
[UIView animateWithDuration:0.3 animations:^{
    self.view.layer.cornerRadius = 20;
    self.view.layer.backgroundColor = [UIColor redColor].CGColor;
}];

// ✅ 直接对 layer 使用隐式动画（绕过 UIView）
[CATransaction begin];
[CATransaction setAnimationDuration:0.5];
[CATransaction setAnimationTimingFunction:[CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut]];
self.view.layer.cornerRadius = 20;
[CATransaction commit];
```

***

## 五、UIView 和 CALayer 的同步机制

```text
UIView.setNeedsDisplay
    ↓
调用 layer.setNeedsDisplay
    ↓
下一个 RunLoop 周期，layer 调用 delegate 的 displayLayer: 或 drawInContext:
    ↓
UIView 实现了 layer 的 delegate，内部调用 drawRect:
    ↓
内容渲染到 layer 的 contents
```

```text
UIView.setNeedsLayout
    ↓
调用 layer.setNeedsLayout
    ↓
下一个 RunLoop 周期，layer 调用 layoutSublayers
    ↓
UIView 实现了 layer 的 delegate，内部调用 layoutSubviews
```

**也就是说：**`drawRect:` **和** `layoutSubviews` **最终都是 CALayer 驱动的。**

***

## 六、实际开发中的常见问题

### 6.1 圆角性能优化

```objc
// ❌ 性能差（触发离屏渲染）
layer.cornerRadius = 10;
layer.masksToBounds = YES;

// ✅ 方案一：只对图片裁剪（图片本身就是圆角的）
// ✅ 方案二：用 CAShapeLayer + UIBezierPath
CAShapeLayer *maskLayer = [CAShapeLayer layer];
maskLayer.path = [UIBezierPath bezierPathWithRoundedRect:layer.bounds
                                            cornerRadius:10].CGPath;
layer.mask = maskLayer;

// ✅ 方案三：shouldRasterize 光栅化（适合静态内容）
layer.cornerRadius = 10;
layer.masksToBounds = YES;
layer.shouldRasterize = YES;
layer.rasterizationScale = [UIScreen mainScreen].scale;
```

### 6.2 阴影性能优化

```objc
// ❌ 性能差（每帧都要计算阴影形状）
layer.shadowOpacity = 0.5;
layer.shadowOffset = CGSizeMake(0, 3);

// ✅ 指定 shadowPath（直接用路径，不需要每帧计算）
layer.shadowPath = [UIBezierPath bezierPathWithRect:layer.bounds].CGPath;
```

### 6.3 离屏渲染触发条件

| 操作                                                | 是否离屏渲染         |
| ------------------------------------------------- | -------------- |
| `cornerRadius + masksToBounds`                    | ✅ 是            |
| `shadow`（无 shadowPath）                            | ✅ 是            |
| `mask`（图层蒙版）                                      | ✅ 是            |
| `allowsGroupOpacity`                              | ✅ 是            |
| `shouldRasterize`                                 | ✅ 是（首次离屏，之后缓存） |
| `shouldRasterize + rasterizationScale ≠ 屏幕 scale` | ✅ 每次都离屏        |

***

## 七、总结速查表

| 概念                   | 要点                                                         |
| -------------------- | ---------------------------------------------------------- |
| **UIView 与 CALayer** | UIView 管交互，CALayer 管显示，layer 是 view 的内部属性                  |
| **为什么分离**            | 跨平台、单一职责、性能（Render Server 直接用 layer 树）                     |
| **Layer 独有属性**       | contents、shadow、3D transform、cornerRadius、border、光栅化       |
| **隐式动画**             | 修改 layer 属性自动产生 0.25s 动画，UIView 禁用了这个行为                    |
| **UIView 禁用隐式动画**    | 作为 layer delegate 返回 `[NSNull null]`，动画块中返回显式动画对象          |
| **drawRect**         | 由 layer 的 display 链路驱动，UIView 是 layer 的 delegate           |
| **圆角 + 阴影冲突**        | masksToBounds 裁剪阴影，用两个 layer 或 shadowPath 解决               |
| **离屏渲染**             | cornerRadius+masksToBounds、shadow、mask、shouldRasterize 会触发 |
