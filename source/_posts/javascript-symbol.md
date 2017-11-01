---
title: 「译」Symbols in ECMAScript 6
date: 2017-01-17 08:35:01
tags:
---

Symbol 是 ES6 中一个新的原始类型，这篇博客解释了它的原理。

## 1. 一个新的原始类型
ES6 带来了一个新的基础类型：Symbol，它可以被作为独一无二的 ID 来使用，，通过调用工厂函数 `Symbol()` 创建一个 symbol（类似把 `String` 作为函数调用时会返回 string 一样）。

`let symbol1 = Symbol();`

`Symbol()` 有一个可选的字符串参数：

```javascript
> let symbol2 = Symbol('symbol2');
> String(symbol2)
'Symbol(symbol2)'
```

`Symbol()` 返回的每一个 symbol 都是独一无二的：

```javascript
> symbol1 === symbol2
false
```

用 typeof 去检测 symbol 可以确认 symbol 是一个原始类型：

```javascript
> typeof symbol1
'symbol'
```

### 2.1 把 symbol 用在属性键上
Symbols 可以用于属性的键：

```javascript
const MY_KEY = Symbol();
let obj = {};

obj[MY_KEY] = 123;
console.log(obj[MY_KEY]); // 123
```

Class 和对象字面量有一个特性叫做 computed property keys：你可以通过把表达式用中括号包裹作为属性的键，下面的代码中我们把 `MY_KEY` 用作对象的键：

```javascript
const MY_KEY = Symbol();
let obj = {
  [MY_KEY]: 123
};
```

方法的定义也可以使用 computed key：

```javascript
const FOO = Symbol();
let obj = {
  [FOO]() {
    return 'bar';
  }
};
console.log(obj[FOO]()); // bar
```

### 2.2 枚举属性
下面我们创建一个对象来检验枚举属性的 API：

```javascript
let obj = {
  [Symbol('my_key')]: 1,
  enum: 2,
  nonEnum: 3
};
Object.defineProperty(obj,
  'nonEnum', { enumerable: false });
```

`Object.getOwnPropertyNames()` 忽略了 symbol 类型的属性键：

```javascript
> Object.getOwnPropertyNames(obj)
['enum', 'nonEnum']
```

`Object.getOwnPropertySymbols()` 忽略了 string 类型的属性键：

```javascript
> Object.getOwnPropertySymbols(obj)
[Symbol(my_key)]
```

`Reflect.ownKeys()` 会返回所有 key：

```javascript
> Reflect.ownKeys(obj)
[Symbol(my_key), 'enum', 'nonEnum']
```

`Object.keys()` 并不像它的名字那样有用，它只作用于字符串类型并且可以枚举的 key：

```javascript
> Object.keys(obj)
['enum']
```

### 2.3 使用 Symbol 作为常量
ES5 中通常使用 string 来作为常量：

```javascript
var COLOR_RED    = 'RED';
var COLOR_ORANGE = 'ORANGE';
var COLOR_YELLOW = 'YELLOW';
var COLOR_GREEN  = 'GREEN';
var COLOR_BLUE   = 'BLUE';
var COLOR_VIOLET = 'VIOLET';
```

然而 string 并不像我们想象的那样不容易冲突，我们来看一个例子：

```javascript
function getComplement(color) {
  switch (color) {
    case COLOR_RED:
      return COLOR_GREEN;
    case COLOR_ORANGE:
      return COLOR_BLUE;
    case COLOR_YELLOW:
      return COLOR_VIOLET;
    case COLOR_GREEN:
      return COLOR_RED;
    case COLOR_BLUE:
      return COLOR_ORANGE;
    case COLOR_VIOLET:
      return COLOR_YELLOW;
    default:
      throw new Exception('Unknown color: '+color);
  }
}
```

值得一提的是你可以在 switch 的 case 里使用任何表达式：

```javascript
function isThree(x) {
  switch (x) {
    case 1 + 1 + 1:
      return true;
    default:
      return false;
  }
}
```

虽然我们现在通过常量（COLOR_RED 等等）来表示而不是 hardcode，但仍会有些问题：

`var MOOD_BLUE = 'BLUE’;`

现在 ‘BLUE’ 不再唯一了而且 MODE_BLUE，把它传进 `getComplement` 函数会返回 `ORANGE` 而不是我们期望的抛错。

下面我们用 Symbol 修复这个问题，同时我们也可以用 ES6 中的 const 让我们创建一个 “真正的” 常量（你不能改变常量的值，但常量本身可能是可变的）。

```javascript
const COLOR_RED    = Symbol();
const COLOR_ORANGE = Symbol();
const COLOR_YELLOW = Symbol();
const COLOR_GREEN  = Symbol();
const COLOR_BLUE   = Symbol();
const COLOR_VIOLET = Symbol();
```

Symbol 返回的每个值都是独一无二的，所以任何一个值都不会出现之前同 `BLUE` 冲突的情况，即使我们用 Symbol 替代了 string，`getComplement()` 的代码依然一行都不用改。

## 3. 将 Symbol 作为属性键名
创建永远不会冲突的 key 在两种场景下非常有用：

- 有其他的对象通过 mixins 将属性设置到同一个对象上
- 保持 meta-level 不同 base-level 属性冲突

### 3.1 Symbol 用作内部属性的 key
Mixin 是暴露出来一组方法，你可以把它拓展到属性或原型上，如果它们的 key 都是 symbol 它们就不会和其他 mixin 方法或者内部方法冲突。

对象的公共方法是可见的，为了使用上方便通常我们更愿意用 string 作为 key，然而内部私有方法只需要暴露给 mixin，它们的 key 适合用 symbol。

Symbol 并不能完全保证私有性，因为遍历出一个对象的 symbol 属性很容易，不过可以保证 key 不冲突通常已经可以满足需求了。如果你的的确确需要确保不被外部访问，你应当使用 `WeakMap` 或者 `closure`：

```javascript
// 每个私有属性对应一个 WeakMap
const PASSWORD = new WeakMap();
class Login {
  constructor(name, password) {
    this.name = name;
    PASSWORD.set(this, password);
  }
  hasPassword(pw) {
    return PASSWORD.get(this) === pw;
  }
}
```

`Login` 的实例就是 WeakMap `PASSWORD` 的 key，`WeakMap` 不会阻止实例被 GC，任何 key 是不存在的 Object 的都会被从 WeakMap 中移除。

使用 symbol 来实现如下：

```javascript
const PASSWORD = Symbol();
class Login {
  constructor(name, password) {
    this.name = name;
    this[PASSWORD] = password;
  }
  hasPassword(pw) {
    return this[PASSWORD] === pw;
  }
}
```

### 3.2 Symbol 用作 meta-level 属性的 key
Symbol 的唯一性使得它很适合用作公共的 meta-level 属性 key，因为 meta-level 属性 key 要避免同普通的 key 冲突。

在 ES6 中，一个对象如果有 Symbol.iterator 这个 key，那么它就是 iterable 的。下面的代码中，obj 就是 iterable 的：

```javascript
let obj = {
  data: [ 'hello', 'world' ],
  [Symbol.iterator]() {
    const self = this;
    let index = 0;
    return {
      next() {
        if (index < self.data.length) {
          return {
            value: self.data[index++]
          };
        } else {
          return { done: true };
        }
      }
    };
  }
};
```

`obj` 的可遍历性允许你用 `for-of` 来遍历它：

`for (let x of obj) console.log(x)`

## 4. 通过 Symbol 跨越执行领域
代码的执行领域指代码所在的上下文，它包括了：全局变量、加载进来的模块等等，虽然代码只存在于一个领域，但是它是有可能访问到另一个领域的代码的。举个例子，在浏览器中每一个标签页都有自己的领域，但是正如下面的代码，执行流可以从一个标签页到另一个：

```html
<head>
  <script>
    function test(arr) {
      var iframe = frames[0];
      // 这段代码和 iframe 的代码在两个不同领域
      // 因此全局变量，比如 Array 是不同的：
      console.log(Array === iframe.Array); // false
      console.log(arr instanceof Array); // false
      console.log(arr instanceof iframe.Array); // true

      // 但是，Symbol 是相同的
      console.log(Symbol.iterator === iframe.Symbol.iterator); // true
    }
  </script>
</head>
<body>
  <iframe srcdoc="<script>window.parent.test([])</script>">
  </iframe>
</body>
```

原因是每个领域都有自己的 Array 副本，因为对象是按引用传递的，所以虽然底层是一样的，但是这些本地的副本都是不等价的。同样的，lib 和客户端代码在每个领域都加载了一次，所以它们的对象也是不等的。

另一面，基础类型 boolean，number 和 string 是按值传递的，所以它们的副本都是相等的。

Symbols 有自己的唯一标识，所以它并不像其他基础类型那样方便的跨领域，这是 symbol 的一个问题：Symbol.iterator 应该是可以跨领域的，如果一个对象在一个领域中是 iterable 的，那么它在另一个领域也应当是同样的。如果 Javascript 引擎提供了跨领域的 Symbol，那么引擎应当可以保证在不同领域中使用的是同一个 Symbol。

对于 lib 我们需要做一点额外的工作，这就涉及到全局 symbol 注册了：注册机制是可以被所有领域使用的，通过 string 指向不同的 symbol，lib 要为每个 symbol 设置一个尽可能独一无二的字符串。为了创建这样的 symbol，lib 可以询问 Symbol 某个 string 指向的 symbol，如果有的话直接返回，没有的话创建一个 symbol。

通过 `Symbol.for()` 从 Symbol 获取注册过的 symbol，获取传给 `Symbol.keyFor()` symbol 获取对应的字符串：

```javascript
> let sym = Symbol.for('Hello everybody!');
> Symbol.keyFor(sym)
'Hello everybody!'
```

不出所料，Javascript 引擎提供的跨领域 Symbol： `Symbol.iterator` 没有注册的 string：

```javascript
> Symbol.keyFor(Symbol.iterator)
undefined
```

## 5. 安全限制
你在使用 Symbol 时可能会遇到两种报错：把 Symbol 作为构造函数使用和强制将 Symbol 转为字符串。

### 5.1 将 Symbol 作为构造函数
你需要通过函数调用的方式创建 symbol，这样的结果就是你很容易错误的把它当作一个构造函数来用，一个 symbol 实例对我们来说并没有什么用处，而且这样做会抛错：

```javascript
> new Symbol()
TypeError: Symbol is not a constructor
```

我们仍然有办法创建一个包了一层的对象，它可以把任何值转成对象，包括 symbol：

```javascript
> let sym = Symbol();
> typeof sym
'symbol'

> let wrapper = Object(sym);
> typeof wrapper
'object'
> wrapper instanceof Symbol
true
```

### 5.2 强制把 symbol 转成 string
既然 string 和 symbol 都可以用作属性 key，你可能会想避免使用者不小心尝试将 symbol 转成 key，举个例子：

`let propertyKey = '__' + anotherPropertyKey;`

ES6 会在隐式转换 string 的时候抛错（底层是通过 ToString）：

```javascript
> var sym = Symbol('My symbol');
> '' + sym
TypeError: Cannot convert a Symbol value to a string
```

不过你还是可以显式的把 symbol 转化为 string：

```javascript
> String(sym)
'Symbol(My symbol)'
> sym.toString()
'Symbol(My symbol)
```

## 6. FAQ
### 6.1 Symbol 属于基础类型还是对象
一方面，symbols 表现得像基础类型，另一方面，symbols 又像对象：

- Symbols 像基础类型：它们被用作属性 key 和抽象概念（比如 `BLUE`）
- 每个 symbol 都有自己的唯一标示，这点让它像对象一样

第二点用对象也可以替代 symbol：

```javascript
const COLOR_RED = Object.freeze({});
···
```

可以通过 `Object.create(null)` 替代 `{}` 进一步压缩对象的体积，需要注意的是，对象不能用作属性的 key。

那么 Symbol 到底是基础类型还是对象呢？它们最终依然是基础类型，有两个原因：

首先，比起对象，Symbol 更像是 string：它们是语言的基础，它们是不可变的并且可以用作属性 key。

其次，symbol 通常被用作属性 key，为了这一点很有必要重新梳理下 Javascript 规范，因为很多对象的功能都多余了：

- 对象可以是其他对象的原型
- 用 proxy 包装一个对象并不能改变它的用途
- Object 可以是内省的，通过：instanceof，Object.keys() 等等

去掉这些功能会让标准更简单，而且据 V8 团队说当处理属性 key 时，基础类型比对象要容易得多。

### 6.2 String 难道不够用吗？
同 string 不同的是，Symbol 是唯一的并且防止了命名冲突，Python 通过 `__iter__` 这样特殊的命名避免冲突，不过如果是 lib 呢？我们可以通过 Symbol 创建一个拓展性强、适应性广的机制，在之后我们要讲到的公共 symbol 这一节中，你会看到 Javascript 本身已经大量应用这一机制了。

## 7. Symbol API
### 7.1 Symbol 函数
- Symbol(description?) -> symbol
会创建一个 symbol，可选参数 `description` 方便 debug，Symbol 的设计并不是一个构造函数，如果你把它当作一个构造函数使用（new Symbol）会抛错

### 7.2 全局 Symbol
有[几个全局的 symbol](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-well-known-symbols) 可以通过 Symbol 的属性访问，它们都是对象的 key，允许你自定义 Javascript 对象的行为。

下面几个基本的操作：

- Symbol.hasInstance（方法）
允许对象 O 自定义 `x instance of` 的行为
- Symbol.toPrimitive（方法）
允许一个对象自定义被强制转换为基础类型时的行为
- Symbol. toStringTag（字符串）
由 `Object.prototype.toString` 调用，默认返回 `'[object '+obj[Symbol.toStringTag]+’]’`

**Iteration：**

- Symbol.iterator（方法）
使一个对象 iterable，返回值是一个 iterator。

**其他：**

- Symbol.unscopables（对象）
当同 `with` 一同使用时，允许对象隐藏某些属性
- Symbol.species（方法）
帮助克隆 `typed arrays` 和RegExp, ArrayBuffer 还有 Promise 的实例。
- Symbol.isConcatSpreadable（布尔值）
标示 `Array.prototype.concat` 应当同对象的元素 concat 或是同对象本身 concat。

### 全局 Symbol 方法
如果你想要一个在所有领域都相等的 symbol，你需要通过注册到全局的 symbol 上来创建：

- Symbol.for(str) -> symbol
通过 `str` 获取对应的 symbol，如果 `str` 尚未注册，那么这个方法会将 `str` 注册在 Symbol 下并返回。

另一个方法允许你反过来通过 symbol 获取注册的 `str`，在序列化 symbol 的时候很有用。

- Symbol.keyFor(sym) -> string
返回 symbol 注册在全局的 `str`，如果没有注册返回 `undefined`

## 8. 延伸阅读
- [Using ECMAScript 6 today](http://www.2ality.com/2014/08/es6-today.html)
- [ECMAScript 6: new OOP features besides classes](http://www.2ality.com/2014/12/es6-oop.html)
- [Iterators and generators in ECMAScript 6](http://www.2ality.com/2013/06/iterators-generators.html)
