---
type: review
---
# React Native 中的列表渲染和性能优化

## 一、核心概念

### 1.1 为什么列表渲染是性能关键

移动端列表可能有成百上千条数据，但屏幕只能显示 5~10 条。如果一次性渲染所有项：

- **内存占用高**：每个列表项都是原生 View，创建和持有都需要内存
- **首屏慢**：即使只显示前几条，也要等全部渲染完成
- **滚动卡顿**：大量 View 参与布局计算和合成

React Native 提供了三种列表组件，解决不同程度的问题。

### 1.2 原生视图与 JS 线程的关系

React Native 的渲染分两步：

1. **JS 线程**：React 组件 → Virtual DOM diff → 生成 UI 指令
2. **Native 线程**：接收指令 → 创建/更新原生视图 → 布局 → 合成

列表性能的瓶颈通常在 **JS 线程**（diff 和指令生成）和 **Bridge 通信**（指令传输）。虚拟化的核心目标就是减少这两处的工作量。

---

## 二、ScrollView

### 2.1 工作原理

```jsx
<ScrollView>
  {data.map(item => <Item key={item.id} data={item} />)}
</ScrollView>
```

ScrollView 的本质：

- 继承自原生 `UIScrollView`（iOS）/ `ScrollView`（Android）
- **一次性渲染所有子元素**，不做虚拟化
- 所有子 View 在初始化时就创建完成，挂载到原生视图树中
- 滚动时只是视口平移，不涉及子 View 的创建/销毁

### 2.2 优缺点

| 维度 | 说明 |
|------|------|
| 优点 | 实现简单、无回收开销、子元素状态天然保留 |
| 缺点 | 数据量大时内存暴涨、首屏渲染慢 |
| 适用 | 少量静态内容（表单、设置页、选项卡内容） |

---

## 三、FlatList

### 3.1 工作原理

```jsx
<FlatList
  data={data}
  renderItem={({ item }) => <Item data={item} />}
  keyExtractor={item => item.id}
/>
```

FlatList 的核心设计：

- 只渲染**屏幕可见区域 + 缓冲区**内的列表项
- 滚动时动态创建新进入缓冲区的项，回收离开缓冲区的项
- 内部维护一个"窗口"，窗口外的项用空白占位或直接不渲染

### 3.2 虚拟化的核心参数

```jsx
<FlatList
  data={data}
  renderItem={renderItem}
  // 以下参数控制虚拟化行为
  initialNumToRender={10}        // 首屏渲染数量
  maxToRenderPerBatch={10}       // 每批次增量渲染数量
  windowSize={21}                // 可视区域倍数（前后各保留多少屏）
  removeClippedSubviews={true}   // 移除屏幕外视图（原生优化）
  updateCellsBatchingPeriod={50} // 两次批量渲染的间隔(ms)
  getItemLayout={(data, index) => ({  // 固定高度优化
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
/>
```

各参数的作用：

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `initialNumToRender` | 10 | 首次渲染的项数，影响首屏时间 |
| `maxToRenderPerBatch` | 10 | 每次增量渲染的项数，影响滚动时的帧率 |
| `windowSize` | 21 | 渲染窗口大小（屏幕高度的倍数），值越大缓冲越多 |
| `removeClippedSubviews` | false | 原生层面移除屏幕外视图，降低内存 |
| `getItemLayout` | undefined | 提供固定高度后跳过布局测量，大幅提升性能 |

### 3.3 FlatList 的性能陷阱

**1. 缺少 `keyExtractor` 或用 index 做 key**

```jsx
// ❌ 错误：用 index 做 key
keyExtractor={(item, index) => String(index)}

// ✅ 正确：用唯一 id
keyExtractor={item => item.id}
```

用 index 做 key 会导致：数据变化时（插入、删除、排序），React 无法正确复用组件，触发大量不必要的重新渲染。

**2. `renderItem` 中创建新函数/对象**

```jsx
// ❌ 每次渲染都创建新的箭头函数和新对象
renderItem={({ item }) => (
  <TouchableOpacity onPress={() => handlePress(item.id)}>
    <Text>{item.name}</Text>
  </TouchableOpacity>
)}

// ✅ 使用 useCallback + memo 缓存
const renderItem = useCallback(({ item }) => (
  <ListItem id={item.id} name={item.name} onPress={handlePress} />
), [handlePress]);

const ListItem = React.memo(({ id, name, onPress }) => (
  <TouchableOpacity onPress={() => onPress(id)}>
    <Text>{name}</Text>
  </TouchableOpacity>
));
```

**3. 没有提供 `getItemLayout`**

没有这个参数时，FlatList 需要对每个项进行布局测量（异步的），无法精确计算滚动位置，导致：
- 滚动条跳动
- `scrollToIndex` 不准确
- 需要等测量完成后才能渲染

**4. 列表项没有使用 `React.memo`**

```jsx
// ❌ 父组件 re-render 时，所有列表项都会 re-render
const Item = ({ data }) => <Text>{data.name}</Text>;

// ✅ 只在 props 变化时 re-render
const Item = React.memo(({ data }) => <Text>{data.name}</Text>);
```

---

## 四、SectionList

### 4.1 工作原理

```jsx
<SectionList
  sections={[
    { title: 'A', data: ['Alice', 'Amy'] },
    { title: 'B', data: ['Bob', 'Beth'] },
  ]}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section }) => <Text>{section.title}</Text>}
  keyExtractor={item => item}
/>
```

SectionList 的特点：

- 继承自 FlatList，**同样支持虚拟化**
- 增加了分组（section）的概念
- 支持 `renderSectionHeader`（组头）和 `renderSectionFooter`（组尾）
- 支持 `stickySectionHeadersEnabled`（组头吸顶）

### 4.2 SectionList vs FlatList

| 维度 | FlatList | SectionList |
|------|----------|-------------|
| 数据结构 | 一维数组 `[{...}, {...}]` | 分组数组 `[{title, data: [...]}]` |
| 组头/组尾 | 不支持 | 支持 `renderSectionHeader/Footer` |
| 吸顶效果 | 不支持 | 支持 `stickySectionHeadersEnabled` |
| 底部组件 | `ListHeaderComponent` / `ListFooterComponent` | 同左 + `SectionSeparatorComponent` |
| 分隔线 | `ItemSeparatorComponent` | 同左 + `SectionSeparatorComponent` |
| 虚拟化 | ✅ | ✅（继承 FlatList） |

---

## 五、FlatList 的虚拟化实现原理

### 5.1 核心思想：窗口化（Windowing）

FlatList 维护一个**虚拟窗口**：

```
|<-------- windowSize × 屏幕高度 -------->|
|     缓冲区（上）  |  可视区域  |  缓冲区（下）  |
|   已渲染但不可见  |  用户可见  |  已渲染但不可见  |
```

- **可视区域**：屏幕实际显示的内容
- **缓冲区**：可视区域上下各保留若干屏的内容，提前渲染，滚动时无需等待
- **窗口外**：不渲染或用空白占位

### 5.2 虚拟化的三个阶段

**阶段一：初始化**

```
1. 调用 getItemLayout（如果有）计算所有项的 offset
2. 渲染 initialNumToRender 个项
3. 如果没有 getItemLayout，异步测量已渲染项的布局
```

**阶段二：滚动更新**

```
1. 监听 onScroll 事件，获取当前滚动位置
2. 根据 windowSize 计算当前渲染窗口的范围 [start, end]
3. 窗口内的新项 → 创建并渲染
4. 窗口外的旧项 → 销毁（或 removeClippedSubviews 隐藏）
5. 批量处理：maxToRenderPerBatch 控制每批数量，updateCellsBatchingPeriod 控制间隔
```

**阶段三：布局测量**

```
1. 没有 getItemLayout 时，每渲染一个项，测量其高度
2. 测量结果缓存，用于后续滚动位置计算
3. 测量是异步的，可能导致滚动条跳动
```

### 5.3 关键内部机制

**VirtualizedList — FlatList 的基类**

FlatList 是 VirtualizedList 的封装。VirtualizedList 的核心逻辑：

```js
// 简化的内部状态
{
  first: 0,           // 当前渲染窗口的起始索引
  last: 10,           // 当前渲染窗口的结束索引
  frames: Map<key, {  // 已测量项的布局信息
    length: number,
    offset: number,
  }>
}
```

**渲染窗口的计算**

```js
// 简化的窗口计算逻辑
const visibleStart = scrollOffset;
const visibleEnd = scrollOffset + viewportLength;
const windowStart = visibleStart - windowSize * viewportLength / 2;
const windowEnd = visibleEnd + windowSize * viewportLength / 2;

// 找到窗口范围内的项
const first = findIndexAtOffset(windowStart);
const last = findIndexAtOffset(windowEnd);
```

---

## 六、FlatList 虚拟化 vs Web 端虚拟列表

### 6.1 Web 端虚拟列表的常见实现

Web 端虚拟列表（如 `react-virtualized`、`react-window`）的典型方案：

**方案一：绝对定位**

```jsx
// 所有项用 absolute 定位，top 值根据 index 计算
<div style={{ height: totalHeight, position: 'relative' }}>
  {visibleItems.map(item => (
    <div style={{
      position: 'absolute',
      top: item.index * itemHeight,
      height: itemHeight,
    }}>
      {item.content}
    </div>
  ))}
</div>
```

**方案二：padding 撑开**

```jsx
// 用上下 padding 模拟完整滚动高度
<div style={{
  paddingTop: aboveHeight,
  paddingBottom: belowHeight,
}}>
  {visibleItems.map(item => item.content)}
</div>
```

**方案三：transform 偏移**

```jsx
// 用 translateY 偏移容器，保持 DOM 结构简单
<div style={{
  transform: `translateY(${offset}px)`,
}}>
  {visibleItems.map(item => item.content)}
</div>
```

### 6.2 核心异同对比

| 维度 | FlatList（RN） | Web 虚拟列表 |
|------|----------------|--------------|
| **渲染引擎** | 原生视图（UIView/AndroidView） | DOM 元素 |
| **虚拟化对象** | 原生 View 的创建/销毁 | DOM 节点的挂载/卸载 |
| **布局计算** | Yoga 引擎（Flexbox） | 浏览器 Layout（Reflow） |
| **滚动容器** | 原生 ScrollView | `overflow: auto` 的 div |
| **高度计算** | 异步测量（bridge 通信） | 同步读取 `offsetHeight` |
| **回收方式** | 销毁或 `removeClippedSubviews` | 移除 DOM 节点 |
| **缓冲区** | `windowSize`（屏幕倍数） | `overscanCount`（项数） |
| **固定高度** | `getItemLayout` 预计算 | 直接传 `itemSize` |
| **动态高度** | 测量后缓存到 `frames` | 预估 + 实测修正 |
| **性能瓶颈** | Bridge 通信、JS 线程 | DOM 操作、Reflow |

### 6.3 关键差异详解

**1. 布局测量是异步 vs 同步**

Web 端读取 DOM 元素高度是同步的：
```js
const height = element.offsetHeight; // 同步，立即返回
```

React Native 测量原生 View 高度是异步的：
```js
view.measure((x, y, width, height) => {
  // 异步回调，经过 Bridge 通信
});
```

这就是为什么 FlatList 在没有 `getItemLayout` 时滚动条会跳动——高度信息是异步获取的。

**2. 回收机制不同**

Web 端虚拟列表：DOM 节点被移除，下次需要时重新创建 DOM。浏览器对 DOM 创建有优化（对象池）。

FlatList：原生 View 被销毁，或通过 `removeClippedSubviews` 标记为不可见（原生层面隐藏，不销毁）。原生 View 的创建比 DOM 更昂贵。

**3. 缓冲区策略不同**

Web 端：`overscanCount` 通常设 5~10 项，因为 DOM 创建便宜。

FlatList：`windowSize` 默认 21（前后各 10 屏），因为原生 View 创建更昂贵，需要更大的缓冲区来保证滚动流畅。

**4. 滚动事件的处理**

Web 端：`scroll` 事件频繁触发，需要用 `requestAnimationFrame` 节流。

FlatList：`onScroll` 事件通过 Bridge 传递，默认已经有节流（`scrollEventThrottle`），但 Bridge 通信本身有延迟。

---

## 七、其他性能优化手段

### 7.1 `removeClippedSubviews`

```jsx
<FlatList
  removeClippedSubviews={true}
  // ...
/>
```

- 原生层面将屏幕外的 View 标记为不可见
- iOS：设置 `clipsToBounds` + 移除 `layer`
- Android：设置 `View.setVisibility(View.GONE)`
- 注意：可能导致某些动画异常

### 7.2 长列表的分页加载

```jsx
<FlatList
  onEndReached={loadMore}
  onEndReachedThreshold={0.5} // 距离底部 50% 时触发
  ListFooterComponent={isLoading ? <ActivityIndicator /> : null}
/>
```

不要一次性加载所有数据，配合分页 API 按需加载。

### 7.3 使用 `FlashList`（Shopify 开源）

```jsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={data}
  renderItem={({ item }) => <Item data={item} />}
  estimatedItemSize={80} // 预估高度
/>
```

FlashList 相比 FlatList 的改进：

| 维度 | FlatList | FlashList |
|------|----------|-----------|
| 回收机制 | 销毁重建 | **原生视图复用**（类似 RecyclerView） |
| 内存 | 随滚动增长 | 基本恒定 |
| 滚动性能 | 批量渲染有延迟 | 更流畅 |
| API | 兼容 FlatList | 几乎完全兼容 |

### 7.4 图片优化

```jsx
// 使用 FastImage 替代 Image
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri: item.imageUrl }}
  resizeMode={FastImage.resizeMode.cover}
  // 内置缓存、渐进式加载
/>
```

### 7.5 避免在列表项中使用匿名函数

```jsx
// ❌ 每次渲染都创建新函数
<Item onPress={() => navigation.navigate('Detail', { id: item.id })} />

// ✅ 使用 useCallback 或将导航逻辑移到组件内部
const Item = React.memo(({ id, navigation }) => {
  const handlePress = useCallback(() => {
    navigation.navigate('Detail', { id });
  }, [id, navigation]);

  return <TouchableOpacity onPress={handlePress}>...</TouchableOpacity>;
});
```

---

## 八、回答问题

### ① ScrollView、FlatList、SectionList 的区别和适用场景

| 维度 | ScrollView | FlatList | SectionList |
|------|------------|----------|-------------|
| 虚拟化 | ❌ 不支持 | ✅ 支持 | ✅ 支持 |
| 数据量 | 少量（< 100） | 大量（成百上千） | 大量且分组 |
| 首屏性能 | 数据多时慢 | 快（只渲染可视区） | 快（继承 FlatList） |
| 内存占用 | 全量渲染，高 | 按需渲染，低 | 按需渲染，低 |
| 分组功能 | 需手动实现 | 需手动实现 | 原生支持 |
| 吸顶效果 | 不支持 | 不支持 | 支持 |
| 适用场景 | 表单、设置页、Tab 内容 | 通讯录、消息列表、Feed 流 | 通讯录按字母分组、设置按类别分组 |

**选型决策树：**

```
需要列表？
├── 数据量少（< 50）且结构简单 → ScrollView
├── 数据量大
│   ├── 不需要分组 → FlatList
│   └── 需要分组 → SectionList
└── 需要水平滚动 + 数据量少 → ScrollView horizontal
    需要水平滚动 + 数据量大 → FlatList horizontal
```

### ② FlatList 的虚拟化实现及与 Web 端虚拟列表的异同

**FlatList 的虚拟化实现：**

FlatList 继承自 `VirtualizedList`，核心是**窗口化（Windowing）**策略：

1. 维护一个渲染窗口，大小由 `windowSize`（默认 21 屏）控制
2. 只渲染窗口内的列表项，窗口外的项不渲染或用空白占位
3. 监听 `onScroll`，根据滚动位置动态调整窗口范围
4. 新进入窗口的项 → 创建并渲染；离开窗口的项 → 销毁
5. 用 `maxToRenderPerBatch` + `updateCellsBatchingPeriod` 控制批量渲染节奏，避免一次性渲染过多导致掉帧

**与 Web 端虚拟列表的异同：**

相同点：
- 核心思想一致：只渲染可视区域 + 缓冲区
- 都需要处理固定高度（直接计算）和动态高度（测量 + 缓存）
- 都面临滚动条跳动、快速滚动白屏等问题

关键差异：
- **布局测量**：Web 同步读取 DOM 高度；RN 异步通过 Bridge 测量原生 View 高度
- **回收机制**：Web 移除 DOM 节点（创建成本低）；RN 销毁原生 View（创建成本高）或用 `removeClippedSubviews` 隐藏
- **缓冲区策略**：Web 用 `overscanCount`（项数，通常 5~10）；RN 用 `windowSize`（屏幕倍数，默认 21），RN 缓冲区更大因为原生 View 创建更贵
- **性能瓶颈**：Web 在 DOM 操作和 Reflow；RN 在 Bridge 通信和 JS 线程
- **滚动事件**：Web 用 `requestAnimationFrame` 节流；RN 用 `scrollEventThrottle` + Bridge 延迟
