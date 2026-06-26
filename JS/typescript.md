---
type: review
---
# TypeScript

## 1. `interface` 和 `type` 有什么区别？

| 对比维度 | `interface` | `type` |
|---|---|---|
| **定义对象** | ✅ 推荐 | ✅ 可以 |
| **联合类型** | ❌ 不支持 | ✅ `type A = string \| number` |
| **交叉类型** | ❌ 不支持（用 extends 继承） | ✅ `type A = B & C` |
| **基本类型别名** | ❌ 不支持 | ✅ `type ID = string` |
| **声明合并** | ✅ 同名 interface 自动合并 | ❌ 同名 type 报错 |
| **extends / implements** | ✅ 都支持 | ✅ 都支持 |
| **Tuple 类型** | ❌ 不支持 | ✅ `type Tuple = [string, number]` |
| **Mapped / Conditional** | ❌ 不支持 | ✅ 原生支持 |

**经验法则**：
- 定义对象结构、API 响应 → 优先用 `interface`（可扩展、可合并）
- 需要联合、交叉、元组、工具类型 → 用 `type`
- 实际项目中两者可混用，团队统一即可

```ts
// interface 声明合并（独有能力）
interface Box { width: number }
interface Box { height: number }
// Box 自动合并为 { width: number; height: number }

// type 做联合类型（interface 做不到）
type Status = 'loading' | 'success' | 'error'
type ID = string | number
```

---

## 2. 什么是泛型？为什么要用泛型？

**泛型（Generics）** 是一种**参数化类型**，把类型当作参数传入，在调用时才确定具体类型。

### 为什么要用泛型？

- **类型安全**：避免 `any` 导致的类型丢失
- **代码复用**：同一段逻辑适用于多种类型
- **类型推导**：使用时自动推导返回类型

### 泛型函数例子

```ts
// 不用泛型 → 丢失类型
function identity1(value: any): any {
  return value  // 返回值类型是 any，调用方不知道具体类型
}

// 用泛型 → 保留类型
function identity<T>(value: T): T {
  return value  // T 是占位符，调用时才确定
}

// 使用
const a = identity<string>('hello')  // a: string
const b = identity(42)               // b: number（自动推导）
```

### 常见泛型用法

```ts
// 泛型接口
interface ApiResponse<T> {
  code: number
  data: T
  message: string
}

// 泛型约束（限制 T 必须有 length 属性）
function getLength<T extends { length: number }>(item: T): number {
  return item.length
}

getLength('hello')     // ✅ string 有 length
getLength([1, 2, 3])   // ✅ array 有 length
getLength(42)           // ❌ number 没有 length
```

---

## 3. `any`、`unknown`、`never`、`void` 分别是什么？

### `any` — 任意类型（关闭类型检查）

```ts
let x: any = 'hello'
x = 42           // ✅ 不报错
x.foo.bar.baz    // ✅ 不报错（运行时会炸）
// 本质：告诉 TS "别管我"，完全放弃类型安全
```

### `unknown` — 未知类型（安全版 any）

```ts
let x: unknown = getData()
x = 42           // ✅ 可以赋值任何类型
x.foo            // ❌ 报错！必须先做类型判断
x.toUpperCase()  // ❌ 报错！

// 必须先收窄类型
if (typeof x === 'string') {
  x.toUpperCase()  // ✅ 安全了
}
```

### `never` — 永远不会有值

```ts
// 场景 1：函数永远不会返回（会抛异常或死循环）
function throwError(msg: string): never {
  throw new Error(msg)
}

function infiniteLoop(): never {
  while (true) {}
}

// 场景 2：穷尽检查（exhaustiveness check）
type Shape = 'circle' | 'square' | 'triangle'

function area(shape: Shape) {
  switch (shape) {
    case 'circle':    return /* ... */
    case 'square':    return /* ... */
    case 'triangle':  return /* ... */
    default:
      const _exhaustive: never = shape  // 如果漏了分支，编译报错
  }
}
```

### `void` — 没有返回值

```ts
function log(msg: string): void {
  console.log(msg)
  // 没有 return，或者 return;
}

// void vs undefined
// void 表示"不关心返回值"，undefined 表示"返回 undefined"
```

### `any` vs `unknown` 核心区别

| 维度 | `any` | `unknown` |
|---|---|---|
| **赋值** | ✅ 任意赋值 | ✅ 任意赋值 |
| **使用** | ✅ 直接操作属性方法 | ❌ 必须先做类型判断 |
| **类型安全** | ❌ 完全放弃 | ✅ 强制安全检查 |
| **传播性** | 污染整个链路 | 被收窄前无法操作 |
| **建议** | 尽量避免 | 作为 `any` 的安全替代 |

**一句话总结**：
- `any`：我知道类型，但我不在乎 → 放弃类型检查
- `unknown`：我不知道类型 → 先检查再用，强制安全
- `never`：这里不可能有值 → 穷尽检查、异常分支
- `void`：函数不返回有意义的值 → 仅用于函数返回类型
