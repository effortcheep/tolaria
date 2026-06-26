---
type: review
relationships:
  Type:
    - "[[review]]"
related_to:
  - "[[CSS]]"
---

# BFC（块级格式化上下文）

## 什么是 BFC

**BFC（Block Formatting Context，块级格式化上下文）** 是 Web 页面中一块**独立的渲染区域**。BFC 内部的元素布局不会影响外部，外部也不会影响内部——就像一个"结界"。

可以理解为：**BFC 是一个隔离的盒子，里面的子元素按规则排版，外面的元素完全感知不到里面的存在。**

---

## 如何触发 BFC

以下任一条件都能创建新的 BFC：

| 触发条件 | 说明 |
|---------|------|
| `display: flow-root` | **最推荐**，专门为此设计，无副作用 |
| `overflow: hidden/auto/scroll` | 不为 `visible` 即可 |
| `display: inline-block` | 转为行内块 |
| `position: absolute/fixed` | 脱离文档流 |
| `float: left/right` | 浮动元素 |
| `display: table-cell/table-caption` | 表格相关 |
| `display: flex/inline-flex` | Flex 容器 |
| `display: grid/inline-grid` | Grid 容器 |
| `contain: layout/content/paint` | CSS Containment |

> **面试重点**：最常考的是 `overflow: hidden` 和 `display: flow-root`。

---

## BFC 的布局规则

### 1. 内部的块级元素垂直排列

BFC 内部的块级盒子会从上到下，一个接一个地垂直排列。

### 2. 同一个 BFC 内，相邻块级盒子的 margin 会折叠

```html
<div class="box1">上</div>
<div class="box2">下</div>
```

```css
.box1 { margin-bottom: 30px; }
.box2 { margin-top: 20px; }
```

**预期间距 50px，实际只有 30px**——这就是 margin 折叠，取较大值。

> **解决方法**：把其中一个盒子放到新的 BFC 中，就能阻止折叠。

### 3. BFC 区域不会与浮动元素重叠

```html
<div class="float">浮动元素</div>
<div class="bfc">BFC 区域</div>
```

```css
.float { float: left; width: 200px; }
.bfc { overflow: hidden; } /* 触发 BFC */
```

普通 `div` 会钻到浮动元素下面，但 **BFC 元素会保持与浮动元素的边界**，不会重叠。这是实现**自适应两栏布局**的经典手段。

### 4. BFC 在计算高度时会包含浮动子元素

```html
<div class="parent">
  <div class="child" style="float: left; height: 100px;">浮动</div>
</div>
```

普通情况下 `parent` 高度为 0（浮动子元素脱离文档流）。给 `parent` 触发 BFC 后，**高度会正确包含浮动子元素**——这就是经典的**清除浮动**方案。

### 5. BFC 是独立的容器，内外互不影响

- 内部元素的定位不会影响外部
- 外部元素的定位不会影响内部
- 浮动子元素不会超出 BFC 边界

---

## BFC 的四大应用场景

### 场景一：解决 margin 折叠

```css
/* 让两个盒子处于不同 BFC，阻止 margin 折叠 */
.wrapper { display: flow-root; }  /* 创建新 BFC */
```

同一个 BFC 内相邻块级元素的 margin 会折叠，放入不同 BFC 即可解决。

### 场景二：清除浮动（父元素高度塌陷）

```css
.parent {
  overflow: hidden;  /* 或 display: flow-root */
  /* BFC 会包含浮动子元素，高度不再塌陷 */
}
```

### 场景三：自适应两栏布局

```css
.sidebar {
  float: left;
  width: 200px;
}
.main {
  overflow: hidden;  /* BFC 不与浮动重叠 */
  /* 自动占据剩余宽度 */
}
```

### 场景四：阻止元素被浮动元素覆盖

```css
.nav-item {
  display: inline-block;  /* 触发 BFC */
  /* 即使旁边有浮动元素，也不会被覆盖 */
}
```

---

## 经典示例：margin 折叠与 BFC 解决

```html
<!-- 折叠的情况 -->
<div class="a" style="margin-bottom: 50px;">A</div>
<div class="b" style="margin-top: 30px;">B</div>
<!-- 实际间距 = max(50, 30) = 50px -->

<!-- 用 BFC 解决 -->
<div class="a" style="margin-bottom: 50px;">A</div>
<div style="display: flow-root;">  <!-- 新 BFC -->
  <div class="b" style="margin-top: 30px;">B</div>
</div>
<!-- 实际间距 = 50 + 30 = 80px，不再折叠 -->
```

---

## BFC 与其它 FC 的区别

| FC | 触发 | 布局方向 | 典型元素 |
|----|------|---------|---------|
| **BFC** | 块级元素 | 垂直排列 | `div`、`p`、`section` |
| **IFC** | 行内元素 | 水平排列 | `span`、`a`、`em` |
| **FFC** | Flex 容器 | 弹性布局 | `display: flex` |
| **GFC** | Grid 容器 | 网格布局 | `display: grid` |

---

## 常见面试问题

### 1. 什么是 BFC？怎么触发？

> BFC 是独立的渲染区域，内部布局不影响外部。通过 `overflow: hidden`、`display: flow-root`、`float`、`position: absolute` 等触发。**最推荐 `display: flow-root`**，无副作用。

### 2. 为什么 `overflow: hidden` 能清除浮动？

> 因为 `overflow: hidden` 触发了 BFC，BFC 规则要求**在计算高度时包含浮动子元素**，所以父元素高度不再塌陷。

### 3. BFC 怎么解决 margin 折叠？

> margin 折叠只在**同一个 BFC 内的相邻块级元素**之间发生。把其中一个元素放进新的 BFC（比如套一个 `display: flow-root` 的容器），它们就不在同一个 BFC 了，margin 自然不会折叠。

### 4. `display: flow-root` 和 `overflow: hidden` 清除浮动有什么区别？

> - `overflow: hidden`：副作用是**裁剪溢出内容**
> - `display: flow-root`：专门为创建 BFC 设计，**没有任何副作用**，语义最清晰

### 5. 浮动元素的高度会被 BFC 计算在内吗？

> 会。BFC 的规则之一就是**计算高度时包含浮动子元素**，这也是清除浮动方案的原理。

### 6. 两个相邻 div，一个 `margin-bottom: 30px`，一个 `margin-top: 20px`，间距是多少？

> **30px**。同一个 BFC 内，相邻块级元素的 margin 会折叠，取较大值。如果想得到 50px，需要让它们处于不同 BFC。

---

## 总结

```
BFC = 独立的渲染结界
  ├── 触发：overflow:hidden / display:flow-root / float / position:absolute ...
  ├── 规则：
  │   ├── 垂直排列
  │   ├── 同 BFC 相邻 margin 折叠
  │   ├── 不与浮动重叠
  │   ├── 计算高度包含浮动子元素
  │   └── 内外隔离
  └── 应用：
      ├── 解决 margin 折叠
      ├── 清除浮动
      ├── 自适应两栏布局
      └── 阻止浮动覆盖
```
