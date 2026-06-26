---
type: review
---
# this 指向

## 核心原则

`this` 的指向**不是在函数定义时确定的，而是在函数调用时确定的**。判断 `this` 的指向，关键看函数的**调用方式**。

---

## 四种绑定规则

### 1. 默认绑定

**独立函数调用**时，`this` 指向全局对象（浏览器中为 `window`，Node 中为 `global`）。严格模式下为 `undefined`。

```js
function foo() {
  console.log(this); // window（非严格模式）
}
foo();
```

### 2. 隐式绑定

函数作为**对象的方法**调用时，`this` 指向调用该方法的对象。

```js
const obj = {
  name: 'Alice',
  greet() {
    console.log(this.name); // 'Alice'，this 指向 obj
  }
};
obj.greet();
```

**隐式丢失**：以下情况会导致 `this` 丢失，退回到默认绑定：

```js
// 情况1：赋值给变量
const fn = obj.greet;
fn(); // this → window，隐式丢失

// 情况2：作为回调传入
setTimeout(obj.greet, 100); // this → window
```

### 3. 显式绑定

通过 `call`、`apply`、`bind` **显式指定** `this`。

```js
function greet() {
  console.log(this.name);
}

const alice = { name: 'Alice' };
const bob = { name: 'Bob' };

// call：立即调用，参数逐个传递
greet.call(alice);    // 'Alice'

// apply：立即调用，参数以数组传递
greet.apply(bob);     // 'Bob'

// bind：返回一个新函数，this 被永久绑定
const greetAlice = greet.bind(alice);
greetAlice();         // 'Alice'
```

**硬绑定**（`bind`）之后的 `this` 无法再被修改：

```js
const bound = greet.bind(alice);
bound.call(bob); // 仍然是 'Alice'，call 无法覆盖 bind
```

### 4. new 绑定

使用 `new` 调用构造函数时，`this` 指向**新创建的实例对象**。

```js
function Person(name) {
  this.name = name; // this → 新创建的实例
}
const p = new Person('Alice');
console.log(p.name); // 'Alice'
```

---

## 优先级（从高到低）

```
new 绑定 > 显式绑定（bind） > 隐式绑定 > 默认绑定
```

---

## 箭头函数的 this

箭头函数**没有自己的 `this`**，它捕获的是**外层作用域的 `this`**，且不可被修改。

```js
const obj = {
  name: 'Alice',
  greet: () => {
    console.log(this.name); // this → 外层作用域（非 obj）
  },
  delayedGreet() {
    setTimeout(() => {
      console.log(this.name); // this → obj（继承外层函数的 this）
    }, 100);
  }
};
```

**关键区别**：
- 普通函数：`this` 在调用时确定
- 箭头函数：`this` 在定义时确定（词法作用域）

**对象字面量 `{}` 不会创建新的作用域**，所以对象中的箭头函数方法，其 `this` 不会指向该对象，而是穿透到对象外层的词法作用域。只有**函数**（普通函数、类、模块等）才会创建新的作用域来决定 `this` 的指向。

**箭头函数不能用 `call`/`apply`/`bind` 修改 `this`**。

---

## 常见场景速查表

| 场景 | this 指向 |
|---|---|
| 普通函数调用 `foo()` | `window` / `undefined`（严格模式） |
| 对象方法 `obj.fn()` | `obj` |
| `call` / `apply` / `bind` | 指定的对象 |
| 构造函数 `new Foo()` | 新创建的实例 |
| 箭头函数 | 外层作用域的 `this` |
| DOM 事件 `el.onclick` | 触发事件的元素 `el` |
| `setTimeout` / 回调函数 | `window`（非严格模式） |
| class 中的方法 | 实例对象 |

---

## 实战易错点

### 1. 回调中 this 丢失

```js
class Timer {
  constructor() {
    this.seconds = 0;
    // ❌ 普通函数，this → window
    setInterval(function () {
      this.seconds++; // 不会生效
    }, 1000);
    // ✅ 箭头函数，this → Timer 实例
    setInterval(() => {
      this.seconds++;
    }, 1000);
  }
}
```

### 2. 对象方法解构后丢失

```js
const obj = {
  name: 'Alice',
  getName() { return this.name; }
};

const { getName } = obj;
console.log(getName()); // undefined（this → window）
// 解决：使用 bind
const getName2 = obj.getName.bind(obj);
```

### 3. class 中传参时 this 丢失

```js
class Button {
  constructor() {
    this.label = 'Click';
  }
  // ❌ 普通方法作为回调会丢失 this
  handleClick() { console.log(this.label); }
  // ✅ 箭头函数属性，this 始终指向实例
  handleClick = () => { console.log(this.label); }
}
```

---

## 总结口诀

> **默认指向 window，隐式看调用对象，显式 call/apply/bind，new 指向新实例，箭头函数看定义处。**
