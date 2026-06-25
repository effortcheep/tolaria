---
type: review
---
# UITableView 性能优化

> **UITableView 滑动卡顿怎么排查和优化？请从Cell复用、异步渲染、高度计算等多个角度来说。**

## 1. 卡顿原因分析

### 1.1 主线程阻塞
- **离屏渲染**：`cornerRadius + masksToBounds`、`shadow`、`mask` 等属性触发 GPU 离屏渲染
- **布局计算**：复杂的 Auto Layout 约束、多次 `layoutSubviews`
- **图片解码**：大图在主线程解码（`UIImage imageNamed` 后首次显示）
- **文本排版**：`NSAttributedString` 的复杂排版、`boundingRect` 计算

### 1.2 帧率目标
- 16.67ms/帧（60fps），超过这个时间就会掉帧
- 120Hz 设备：8.33ms/帧

---

## 2. Cell 复用优化

### 2.1 复用机制原理
```objc
// 注册复用标识符
[self.tableView registerClass:[MyCell class] forCellReuseIdentifier:@"CellID"];

// 从复用队列获取
UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"CellID" forIndexPath:indexPath];
```

> **复用池容量**：默认只保留当前屏幕可见 Cell 数量 + 2 个缓冲（上下各预加载一个），超出范围的 Cell 会被移出复用池。

### 2.2 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Cell 闪烁 | `prepareForReuse` 未清空旧数据 | 重置图片、隐藏占位视图 |
| 数据错位 | 异步回调未校验 indexPath | 回调时对比当前 indexPath |
| 复用失效 | Cell 注册错误或标识符不一致 | 统一注册方式 |

### 2.3 最佳实践
```objc
- (void)prepareForReuse {
    [super prepareForReuse];
    self.imageView.image = nil;  // 清空旧图
    self.label.text = nil;       // 清空旧文本
}
```

---

## 3. 异步渲染

### 3.1 核心思路
将耗时的绘制操作从主线程移到子线程：

```objc
// 子线程绘制
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    UIGraphicsBeginImageContextWithOptions(size, NO, scale);
    // 绘制操作
    CGContextRef context = UIGraphicsGetCurrentContext();
    // ... 复杂绘制
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    dispatch_async(dispatch_get_main_queue(), ^{
        self.layer.contents = (__bridge id)image.CGImage;
    });
});
```

### 3.2 图片异步加载
```objc
// 使用 SDWebImage 或自定义方案
[cell.imageView sd_setImageWithURL:url
                  placeholderImage:placeholder
                         options:SDWebImageAvoidAutoSetImage
                        completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
    if (image) {
        cell.imageView.image = image;
        [cell setNeedsLayout]; // 触发布局更新
    }
}];
```

### 3.3 文本异步排版
```objc
// 使用 TextKit 2 或 YYText 的异步排版
YYLabel *label = [YYLabel new];
label.displaysAsynchronously = YES;  // 开启异步渲染
label.fadeOnAsynchronouslyDisplay = NO; // 关闭淡入动画
```

---

## 4. 高度计算优化

### 4.1 问题根源
- `heightForRowAtIndexPath` 在滑动时频繁调用（每个 Cell 都会调用）
- 复杂 Auto Layout 计算耗时

### 4.2 缓存高度
```objc
// 使用字典缓存高度
@property (nonatomic, strong) NSMutableDictionary *heightCache;

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    NSString *key = [NSString stringWithFormat:@"%ld-%ld", (long)indexPath.section, (long)indexPath.row];
    NSNumber *cachedHeight = self.heightCache[key];
    if (cachedHeight) {
        return cachedHeight.floatValue;
    }
    
    CGFloat height = [self calculateHeightForCellAtIndexPath:indexPath];
    self.heightCache[key] = @(height);
    return height;
}
```

### 4.3 预计算高度
```objc
// 在数据源返回后立即计算
- (void)configureWithModel:(MyModel *)model {
    self.nameLabel.text = model.name;
    self.detailLabel.text = model.detail;
    
    // 预计算布局
    [self layoutIfNeeded];
    model.cachedHeight = CGRectGetMaxY(self.contentView.frame);
}
```

### 4.4 Auto Layout 优化
```objc
// 使用 estimatedRowHeight + 自动计算
self.tableView.estimatedRowHeight = 80;
self.tableView.rowHeight = UITableViewAutomaticDimension;

// 避免复杂约束：减少嵌套、使用 frame 布局替代
```

---

## 5. 离屏渲染优化

### 5.1 触发离屏渲染的操作
```objc
// ❌ 会触发离屏渲染的组合
view.layer.cornerRadius = 10;
view.layer.masksToBounds = YES;    // 这两个一起才触发
// 因为裁剪需要先渲染到新缓冲区再裁剪
view.layer.shadowOffset = CGSizeMake(0, 3);
view.layer.shadowOpacity = 0.5;

// ✅ 优化方案
// 1. 使用贝塞尔曲线裁剪
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:10];
CAShapeLayer *maskLayer = [CAShapeLayer new];
maskLayer.path = path.CGPath;
view.layer.mask = maskLayer;

// 2. 预渲染圆角图片
UIGraphicsBeginImageContextWithOptions(size, NO, 0);
UIBezierPath *clipPath = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:10];
[clipPath addClip];
[image drawInRect:rect];
UIImage *roundedImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
```

### 5.2 Shadow 优化
```objc
// 指定 shadowPath 避免离屏渲染
view.layer.shadowPath = [UIBezierPath bezierPathWithRoundedRect:view.bounds cornerRadius:10].CGPath;
```

---

## 6. 其他优化手段

### 6.1 减少 Cell 层级
```objc
// ❌ 深层级
UIView -> UIView -> UIView -> UILabel

// ✅ 扁平化
UIView -> UILabel (直接添加到 contentView)
```

### 6.2 预加载与预取
```objc
// iOS 10+ 使用 prefetchDataSource
self.tableView.prefetchDataSource = self;

- (void)tableView:(UITableView *)tableView prefetchRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths {
    // 预加载数据、图片
    for (NSIndexPath *indexPath in indexPaths) {
        [self prefetchDataForIndexPath:indexPath];
    }
}
```

### 6.3 局部刷新
```objc
// ❌ 全量刷新
[self.tableView reloadData];

// ✅ 局部刷新
[self.tableView reloadRowsAtIndexPaths:@[indexPath]
                      withRowAnimation:UITableViewRowAnimationNone];
```

### 6.4 异步绘制（drawRect）
```objc
// 重写 drawRect 在子线程绘制
- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    // 自定义绘制
    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
    CGContextFillRect(context, rect);
}
```

---

## 7. 排查工具

### 7.1 Instruments
- **Time Profiler**：分析 CPU 耗时
- **Core Animation**：检测离屏渲染（Color Off-screen Rendered）
- **GPU Driver**：GPU 使用率

### 7.2 Xcode 调试
```objc
// 开启帧率显示
// Debug -> View Debugging -> Rendering -> Color Blended Layers
// Debug -> View Debugging -> Rendering -> Color Off-screen Rendered
```

### 7.3 代码监控
```objc
// 监控主线程卡顿
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
dispatch_async(dispatch_get_main_queue(), ^{
    dispatch_semaphore_signal(semaphore);
});
// 超过 16ms 未返回说明卡顿
```

---

## 8. 总结清单

| 优化方向 | 具体措施 | 预期效果 |
|----------|----------|----------|
| Cell 复用 | 正确使用 reuseIdentifier | 减少内存占用 |
| 异步渲染 | 图片/文本子线程处理 | 降低主线程压力 |
| 高度缓存 | 缓存 + 预计算 | 减少重复计算 |
| 离屏渲染 | shadowPath / 贝塞尔裁剪 | 降低 GPU 负担 |
| 层级优化 | 扁平化视图层级 | 减少布局计算 |
| 预加载 | prefetchDataSource | 提前准备数据 |
| 局部刷新 | reloadRowsAtIndexPaths | 避免全量刷新 |