---
type: review
---
# CSS 定位与居中方案

## 1. `position` 的五个值

### static（默认值）
- 元素按照正常文档流排列，不受 `top`、`right`、`bottom`、`left` 和 `z-index` 影响。

### relative（相对定位）
- 元素**相对自身原本位置**进行偏移。
- 原本占据的空间**保留**，不会影响其他元素的布局。
- 可作为子元素 `absolute` 定位的参照物（包含块）。

### absolute（绝对定位）
- 元素**脱离文档流**，不占据原来的空间。
- 相对于**最近的非 static 定位祖先元素**进行定位；如果没有这样的祖先，则相对于初始包含块（`<html>`）。
- 可配合 `top`、`right`、`bottom`、`left` 精确定位。

### fixed（固定定位）
- 元素脱离文档流，相对于**浏览器视口（viewport）**定位。
- 滚动页面时位置**固定不动**。
- 常用于导航栏、回到顶部按钮等。

### sticky（粘性定位）
- 是 `relative` 和 `fixed` 的混合体。
- 元素在跨越特定阈值前按 `relative` 排列，达到阈值后变为 `fixed` 效果。
- 必须指定 `top`、`right`、`bottom`、`left` 中的至少一个。
- 常用于表头吸顶效果。

### `relative` 和 `absolute` 的区别

| | relative | absolute |
|---|---|---|
| 是否脱离文档流 | 否，保留原位 | 是，脱离文档流 |
| 定位参照 | 自身原来的位置 | 最近的非 static 定位祖先 |
| 对其他元素的影响 | 无影响（原位保留） | 后续元素会上移填补空位 |
| 是否创建新的包含块 | 否（除非配合 transform 等） | 是 |

### 五个 position 值的文档流对比

| position 值 | 是否脱离文档流 | 占据原始空间 |
|---|---|---|
| **static** | 否 | 是 |
| **relative** | 否 | 是（原位保留） |
| **absolute** | 是 | 否（后续元素填补） |
| **fixed** | 是 | 否（后续元素填补） |
| **sticky** | 否（未触发阈值时） | 是 |

---

## 2. 水平垂直居中方案

### 方案一：Flexbox（推荐）

```css
.parent {
  display: flex;
  justify-content: center; /* 水平居中 */
  align-items: center;     /* 垂直居中 */
}
```

### 方案二：Grid 布局

```css
.parent {
  display: grid;
  place-items: center; /* 同时水平垂直居中 */
}
```

### 方案三：绝对定位 + transform

```css
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

### 方案四：绝对定位 + margin: auto

```css
.child {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  margin: auto;
  width: 200px;  /* 需要指定宽高 */
  height: 100px;
}
```

### 方案五：Grid + place-items 替代写法

```css
.parent {
  display: grid;
  justify-items: center;
  align-items: center;
}
```

---

## 总结速记

- **Flexbox** 是最常用、兼容性好的居中方案。
- **Grid** 代码最简洁，适合二维布局。
- **绝对定位 + transform** 适用于不知道元素宽高的场景。
- **绝对定位 + margin: auto** 需要已知宽高。
