---
type: review
---

# React Native 的导航（React Navigation）

## 一、核心概念

### 为什么需要导航库

- RN 没有浏览器的 URL 机制，无法用 `<a>` 标签跳转
- 原生应用的导航是"栈"模型：页面入栈（push）、出栈（pop）、替换（replace）
- 导航库负责管理页面栈、切换动画、参数传递、深链接（Deep Linking）

### React Navigation 的定位

- React Navigation 是 RN 官方推荐的导航方案
- 核心包：`@react-navigation/native`（基础）+ 各种导航器包
- 纯 JS 实现（不依赖原生导航控制器），通过自定义 Animated 驱动转场动画

### 导航器（Navigator）的本质

- 导航器是一个 React 组件，管理一组页面（Screen）的状态
- 内部维护一个"路由状态"对象（`{ routes, index }`），描述当前有哪些页面、哪个在最前面
- 导航器之间可以嵌套：Stack 里面放 Tab，Tab 里面再放 Stack

---

## 二、导航器类型详解

### 1. Stack Navigator（堆栈导航器）

```bash
npm install @react-navigation/native-stack
# 或
npm install @react-navigation/stack
```

- **行为**：页面像卡片一样堆叠，新页面从右侧推入，返回时向右滑出
- **API**：`navigation.push('Screen')` / `navigation.pop()` / `navigation.goBack()`
- **适用场景**：绝大多数页面跳转，详情页、表单页、设置子页
- **两个版本**：
  - `native-stack`：使用原生导航控制器（iOS UINavigationController / Android Fragment），性能更好，支持原生转场手势
  - `stack`：纯 JS 实现，动画可定制性更强，但性能略差

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function RootStack() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
      <Stack.Screen name="Settings" component={SettingsScreen} />
    </Stack.Navigator>
  );
}
```

### 2. Tab Navigator（标签导航器）

```bash
npm install @react-navigation/bottom-tabs
```

- **行为**：底部（或顶部）标签栏切换页面，页面之间互相独立，不会互相推栈
- **API**：`navigation.navigate('TabName')`
- **适用场景**：应用的主导航结构（首页、发现、消息、我的）
- **特点**：切换 Tab 时页面默认不销毁（保持状态），除非配置 `unmountOnBlur: true`

```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MainTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Discover" component={DiscoverScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

### 3. Drawer Navigator（抽屉导航器）

```bash
npm install @react-navigation/drawer
```

- **行为**：从屏幕边缘滑出一个侧边抽屉面板
- **API**：`navigation.openDrawer()` / `navigation.closeDrawer()` / `navigation.toggleDrawer()`
- **适用场景**：层级较多的应用主菜单（如 Gmail、微信读书的侧边栏）

```tsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function AppDrawer() {
  return (
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={HomeScreen} />
      <Drawer.Screen name="Settings" component={SettingsScreen} />
    </Drawer.Navigator>
  );
}
```

### 4. Material Top Tab Navigator（顶部标签导航器）

```bash
npm install @react-navigation/material-top-tabs
```

- **行为**：顶部 Tab 栏，支持手势左右滑动切换
- **适用场景**：内容分类浏览（如微信的聊天/通讯录/发现，微博的关注/推荐）
- **特点**：依赖 `react-native-tab-view` + `react-native-pager-view`

### 5. Native Stack vs JS Stack

| 对比维度 | `native-stack` | `stack`（JS） |
| --- | --- | --- |
| 转场动画 | 原生系统动画 | JS Animated 动画 |
| 手势返回 | 原生手势，跟手流畅 | JS 手势，可能卡顿 |
| 大标题（iOS） | 支持原生大标题 | 不支持 |
| 自定义程度 | 受限于原生 API | 完全自定义 |
| 性能 | 更好（不走 JS 线程） | 较差（JS 驱动） |
| 推荐 | ✅ 首选 | 需要高度自定义时 |

---

## 三、导航器嵌套

导航器嵌套是实际开发的核心模式：

```tsx
function App() {
  return (
    <Stack.Navigator>
      {/* 主 Tab 在 Stack 里 */}
      <Stack.Screen name="MainTabs" component={MainTabs} options={{ headerShown: false }} />
      {/* 详情页在 Stack 里，覆盖在 Tab 之上 */}
      <Stack.Screen name="Detail" component={DetailScreen} />
      <Stack.Screen name="Login" component={LoginScreen} />
    </Stack.Navigator>
  );
}

function MainTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeStack} options={{ headerShown: false }} />
      <Tab.Screen name="Profile" component={ProfileStack} options={{ headerShown: false }} />
    </Tab.Navigator>
  );
}

function HomeStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="HomeList" component={HomeListScreen} />
      <Stack.Screen name="HomeDetail" component={HomeDetailScreen} />
    </Stack.Navigator>
  );
}
```

**嵌套时的导航行为**：
- `navigation.navigate('Home')` — 跳到 Tab 名（切换 Tab）
- `navigation.navigate('HomeDetail')` — 如果当前在 Home Tab 的 Stack 里，会 push；如果在其他 Tab，会先切到 Home Tab 再 push
- `navigation.push('HomeDetail')` — 始终 push 一个新页面（即使已经在 HomeDetail）

---

## 四、页面之间传递参数

### 4.1 跳转时传参

```tsx
// 方式一：navigate 传参
navigation.navigate('Detail', {
  id: 123,
  title: '文章标题',
});

// 方式二：push 传参
navigation.push('Detail', { id: 456 });
```

### 4.2 接收参数

```tsx
function DetailScreen({ route }) {
  // route.params 就是传过来的参数对象
  const { id, title } = route.params;

  return (
    <View>
      <Text>ID: {id}</Text>
      <Text>标题: {title}</Text>
    </View>
  );
}
```

### 4.3 TypeScript 类型声明

```tsx
// 定义路由参数类型
type RootStackParamList = {
  Home: undefined;                    // 无参数
  Detail: { id: number; title: string }; // 有参数
  Login: { redirect?: string };       // 可选参数
};

// 使用
const Stack = createNativeStackNavigator<RootStackParamList>();

// 在组件中获得类型提示
function DetailScreen({ route, navigation }: NativeStackScreenProps<RootStackParamList, 'Detail'>) {
  const { id, title } = route.params; // ✅ 有类型提示
  navigation.navigate('Detail', { id: 789, title: '新标题' }); // ✅ 参数类型检查
}
```

### 4.4 通过 Screen options 配置初始参数

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  initialParams={{ id: 0 }}  // 默认参数，当 navigate 不传参时使用
/>
```

### 4.5 参数的安全访问

```tsx
function DetailScreen({ route }) {
  // ❌ 不安全：params 可能为空
  const id = route.params.id;

  // ✅ 安全：提供默认值
  const { id = 0, title = '默认标题' } = route.params ?? {};
}
```

---

## 五、页面之间返回数据

### 5.1 React Navigation 6 的方式：通过 `navigate` 回调

React Navigation 6 没有内置的 "goBack(result)" API，推荐的模式是：

**方式一：通过全局状态（推荐）**

```tsx
// 使用 Zustand / Redux / Context
function DetailScreen({ navigation }) {
  const setSelectedItem = useStore(state => state.setSelectedItem);

  function handleSelect(item) {
    setSelectedItem(item);        // 写入全局状态
    navigation.goBack();          // 返回上一页
  }
}

function HomeScreen() {
  const selectedItem = useStore(state => state.selectedItem); // 上一页自动拿到
}
```

**方式二：通过 route.params 回传（反向传参）**

```tsx
// 从 Home 跳到 Detail，传一个回调函数
function HomeScreen({ navigation }) {
  const [result, setResult] = useState(null);

  navigation.navigate('Detail', {
    onSelect: (item) => setResult(item),  // 传回调函数
  });
}

// Detail 页面调用回调
function DetailScreen({ route, navigation }) {
  const { onSelect } = route.params;

  function handlePress(item) {
    onSelect(item);        // 调用回调
    navigation.goBack();   // 返回
  }
}
```

**方式三：使用 `navigation.setParams` 反向修改参数**

```tsx
// Detail 页面修改 Home 页面的 params
function DetailScreen({ navigation }) {
  function handleConfirm(data) {
    navigation.navigate('Home', { result: data });
    // 或者
    navigation.setParams({ result: data });
  }
}

// Home 页面监听 params 变化
function HomeScreen({ route }) {
  useEffect(() => {
    if (route.params?.result) {
      // 处理返回的数据
    }
  }, [route.params?.result]);
}
```

### 5.2 典型场景：拍照/选图后返回结果

```tsx
function CameraScreen({ navigation }) {
  function onCapture(photo) {
    // 返回上一页并传递照片数据
    navigation.navigate('Publish', { photo: photo.uri });
  }
}

function PublishScreen({ route }) {
  const photo = route.params?.photo; // 拿到照片
}
```

---

## 六、navigation 对象的常用方法

| 方法 | 说明 |
| --- | --- |
| `navigate('Name', params)` | 跳转到指定页面（栈中有则回到该页面，否则 push） |
| `push('Name', params)` | 始终 push 新页面（即使栈中有同名页面） |
| `pop()` | 弹出当前页面（回到上一页） |
| `popToTop()` | 弹出到栈顶（回到第一个页面） |
| `goBack()` | 返回上一页（跨导航器也可用） |
| `replace('Name', params)` | 替换当前页面（不会新增栈项） |
| `reset(state)` | 重置整个导航状态 |
| `setParams(params)` | 修改当前页面的 params |
| `setOptions(options)` | 动态修改当前页面的 header 配置 |

---

## 七、Deep Linking（深链接）

将 URL 映射到导航状态，实现从外部链接直接打开应用内指定页面：

```tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: {
        screens: {
          HomeList: 'home',
          HomeDetail: 'home/:id',
        },
      },
      Detail: 'detail/:id',
      Login: 'login',
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking}>
      <RootStack />
    </NavigationContainer>
  );
}
```

访问 `myapp://detail/123` → 自动导航到 Detail 页面，`route.params.id` 为 `'123'`。

---

## 八、回答问题

### ① React Navigation 有哪几种导航器类型？分别适用什么场景？

| 导航器 | 包名 | 适用场景 |
| --- | --- | --- |
| **Stack** | `@react-navigation/native-stack` | 页面跳转、详情页、表单流程，**最常用** |
| **Bottom Tab** | `@react-navigation/bottom-tabs` | 应用主导航框架（首页/发现/我的） |
| **Drawer** | `@react-navigation/drawer` | 侧边菜单，层级较多的应用 |
| **Material Top Tab** | `@react-navigation/material-top-tabs` | 顶部内容分类（聊天/通讯录/发现） |

实际开发中通常是 **Stack + Tab 嵌套**：外层 Stack 管理登录/主页/详情的跳转，内层 Tab 管理主页的几个 Tab 页。每个 Tab 内部还可以嵌套自己的 Stack。

### ② React Navigation 中页面之间怎么传递参数和返回数据？

**传递参数**：`navigate('Screen', { key: value })` 发送，`route.params` 接收。TypeScript 中通过 `RootStackParamList` 泛型声明参数类型。

**返回数据**：React Navigation 没有内置的"带回调返回"API，三种常用方式：
1. **全局状态**（Zustand/Redux）— 跳转前写入状态，返回后自动读取，最推荐
2. **回调函数**（通过 params 传递）— 简单场景直接把 `setState` 传给下一页
3. **setParams** — 下一页通过 `navigate('Home', { result })` 把数据塞回上一页的 params，上一页通过 `useEffect` 监听
