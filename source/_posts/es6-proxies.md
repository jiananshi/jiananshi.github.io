---
title: 「译」通过 ECMAScript 6 proxies 进行元编程
date: 2017-02-13 21:39:15
tags: javascript es6 proxy
---

> 介绍 Proxy 的一篇好文，原文来自 2ality，原文挺长的，拖了很久才成稿。原文地址：[Meta programming with ECMAScript 6 proxies](http://www.2ality.com/2014/12/es6-proxies.html)

这篇博客介绍 ES6 的一个特性：Proxies，Proxies 允许开发者拦截并自定义一个对象（比如获取属性），这属于元编程的特性。

本文的代码会使用到其它一些 ES6 特性，查看 [Using ECMAScript 6 today](http://www.2ality.com/2014/08/es6-today.html) 来全面了解 ES6 都有哪些功能。

在我们学习 Proxies 以及解释 Proxies 的用处之前，我们首先需要理解 —— 什么是元编程？

## 编程和元编程
编程有以下两种级别（原文为 level，这里翻译为级别，作者是指层级而非孰优孰劣）：

- 基础等级（也叫做应用层）：代码主要处理用户的输入
- 元编程：处理应用层代码的代码

应用层代码和元编程可以是两种截然不同的编程语言，下面的代码中，元编程是 Javascript，应用层则是 Java：

```javascript
let str = 'Hello' + '!'.repeat(3);
console.log('System.out.println("'+str+'")');
```

* 元编程有很多种形式，上面的代码中我们把 Java 代码打印到了控制台，下面的代码中我会演示如何使用 Javascript 同时编写应用层代码和元编程，一个非常经典的例子就是 `eval`，`eval` 可以让你实时编译 Javascript 代码。你真正需要用到 eval 的场景 [非常少](http://speakingjs.com/es5/ch23.html#_legitimate_use_cases)，下面的例子中我们用 eval 来计算 `5 + 2`：

```javascript
eval('5 + 2');
7
```

Javascript 其他的一些操作咋看起来不像是元编程，但是仔细思考的话你会另有发现：

```javascript
// 应用层
let obj = {
  hello() {
    console.log('Hello!');
  }
};
    
// 元编程
for (let key of Object.keys(obj)) {
  console.log(key);
}
```

这段代码在运行的同时也在检查自己的结构（obj），它看起来不像是元编程的主要原因是 Javascript 中编程结构和数据结构的界限非常模糊不清，所有的 `Object.*` 方法都可以被认为是元编程。

### 1.1 元编程的种类
反射式元编程的意思是程序自己处理自己，Kiczales 在 [这篇文章](http://mitpress.mit.edu/books/art-metaobject-protocol) 中探讨了反射式元编程的三种类型：

- 内省性：你对程序结构只有只读权限
- 自我修改性：你可以改变结构
- 干预性：你可以重新定义操作的语义

下面我们来看几个例子：

**例子：内省性**：`Object.keys()` 证明了内省（具体例子在前面的代码中）。

**例子：自我修改性**：下面的代码中，函数 `moveProperty` 将属性从一个对象移到另一个对象上，它通过使用中括号语法获取属性、赋值操作符和 `delete` 操作符证明了自我修改性。

```javascript
function moveProperty(source, propertyName, target) {
  target[propertyName] = source[propertyName];
  delete source[propertyName];
}
```

使用 `moveProperty()`：

```javascript
> let obj1 = { prop: 'abc' };
> let obj2 = {};
> moveProperty(obj1, 'prop', obj2);
    
> obj1
{}
> obj2
{ prop: 'abc' }
```

Javascript 目前为止并不具有干预性，Proxies 因此而被创建。

## Proxies 总览
ES6 Proxies 为 Javascript 提供了干预的功能，你可以对对象 `obj` 进行很多操作，举个例子：

- 获取属性 `prop`（通过 `obj.prop`）
- 列举出所有可枚举的属性（通过 `Object.keys(obj)`）

Proxies 是一种独特的对象，通过它你可以为一些操作自定义实现，一个proxy 实例的创建需要两个参数：

- handler: 每一个操作都有一个对应的 handler 方法，这种干预其他操作的方法被称为 trap（这个词是从操作系统领域借鉴来的）。
- target: 如果 handler 方法没有干预操作，那么它就是直接在 target 上应用，换句话说 proxy 在 target 外包了一层。

下面的代码中 handler 干预了操作 get。

```javascript
let target = {};
let handler = {
  get(target, propKey, receiver) {
    console.log('get ' + propKey);
    return 123;
  }
};
let proxy = new Proxy(target, handler);
```

当我们尝试获取属性 `foo` 时，proxy 劫持了这个操作：

```javascript
> proxy.foo
get foo
123
```

handler 并没有实现 set 方法，所以如果我们尝试去设置 `proxy.bar`，那么最终 target.bar 会被设置。

```javascript
> proxy.bar = 'abc';
> target.bar
'abc'
```

### 2.1 应用在函数上的 trap
如果 target 是一个函数，那么我们可以额外增加两个劫持对象：

- apply: 进行一次函数调用，通过 `proxy(…)`，`proxy.call(…)`，`proxy.apply(…)` 触发
- construct: 进行一次构造函数调用，通过 `new proxy(…)` 触发

只在函数上提供这两个 trap 的原因很简单：在其他对象上你也用不到这两个操作。

### 2.2 可执行的 proxies
ES6 允许你创建可执行的 proxy：

`let {proxy, revoke} = Proxy.revocable(target, handler);`

在代码的左值我们使用了 [解构](http://www.2ality.com/2014/06/es6-multiple-return-values.html) 来获取 `Proxy.revocable()` 返回的属性：proxy 和 revoke。

在你第一次调用 revoke 过后，对 proxy 进行的任何操作都会返回一个 TypeError，后续继续调用 revoke 也不会有任何边际影响。

```javascript
let target = {}; // 创建一个空对象
let handler = {}; // 不劫持任何属性
let {proxy, revoke} = Proxy.revocable(target, handler);
    
proxy.foo = 123;
console.log(proxy.foo); // 123
    
revoke();
    
console.log(proxy.foo); // TypeError: Revoked
```

### 2.3 Proxies 用作原型
proxy 的 proto 对象可以作为对象 obj 的原型，某些在 `obj` 上执行的操作可能会一直沿着原型链想上查找到 `proto`，比如 get。

```javascript
let proto = new Proxy({}, {
  get(target, propertyKey, receiver) {
    console.log('GET '+propertyKey);
    return target[propertyKey];
  }
});
    
let obj = Object.create(proto);
obj.bla; // Output: GET bla
```

属性 `bla` 并没有在 `obj` 对象上，因此原型链一直查找到了 `proto` 并触发了 get 方法。除此之外还有很多其他会影响到原型的方法，我把它们在这篇博客的最后都列举出来了。

### 2.4 操作转发
有的时候你可能想在转发操作的时候额外做一些事情，举个例子，一个 handler 劫持并打印出所有的操作，却不影响它们应用在目标对象上。

```javascript
let handler = {
  deleteProperty(target, propKey) {
    console.log('DELETE ' + propKey);
    return delete target[propKey];
  },
  has(target, propKey) {
    console.log('HAS ' + propKey);
    return propKey in target;
  }
}
```

对于每个 trap，我们首先输出操作名然后手动转发它，ES6 有一个特性可以帮助我们完成转发叫做 `Reflect`。瞎 main我们使用 Reflect 重写上面的代码：

```javascript
let handler = {
  deleteProperty(target, propKey) {
    console.log('DELETE ' + propKey);
    return Reflect.deleteProperty(target, propKey);
  },
  has(target, propKey) {
    console.log('HAS ' + propKey);
    return Reflect.has(target, propKey);
  }
}
```

接着我们用 proxy 来实现这个 handler：

```javascript
let handler = new Proxy({}, {
  get(target, trapName, receiver) {
    // 返回 trapName 作为 handler 函数的名字
    return function (...args) {
      // 取出 args[0]
      console.log(trapName.toUpperCase()+' '+args.slice(1));
      // 转发操作
      return Reflect[trapName](...args);
    }
  }
});
```

对于每个 trap，proxy 通过 get 操作获取 trapName 对应的函数，现在，所有的函数都可以通过一个 get 操作来实现，这也是 proxy API 设计的目标之一。

下面让我们来使用这个基于 proxy 的函数：

```javascript
> let target = {};
> let proxy = new Proxy(target, handler);
> proxy.foo = 123;
SET foo,123,[object Object]
> proxy.foo
GET foo,[object Object]
123
```

下面的代码证实了 set 方法确实将属性写到 target 上了：

```javascript
> target.foo
123
```

## 3. Proxy 的使用场景
### 3.1 用纯 JS 实现 DOM
浏览器 Document Object Model（DOM）通常是混合使用 C++ 和 JS 实现的，单纯通过 JS 实现的好处在于：
- 模拟一个浏览器环境，[jsdom](https://github.com/tmpvar/jsdom) 就是一个在 nodejs 中操作 HTML 的库。
- 提高 DOM 性能（在 JS 和 C++ 中切换是有时间成本的）

不过标准的 DOM 有些特性很难用 JS 实现，举个例子，大部分 DOM 都可以在改变时实时更新在视图上，而 JS 实现的话效率就并没有那么高。为 JS 提供 proxy 的一大原因就是希望可以更高效的实现 DOM。

### 3.2 访问 RESTful Web 服务
proxy 可以创建一个可以执行任何方法的对象，下面的例子中 `createWebService` 就创建了一个这样的对象 `service`，在 `service` 上执行一个方法会返回方法同名的资源。

```javascript
let service = createWebService('http://example.com/data');
// Read JSON data in http://example.com/data/employees
service.employees().then(json => {
  let employees = JSON.parse(json);
  ···
});
```

下面的代码是 ES5 实现的 `createService`，因为不能使用 proxy 所以我们必须预先知道 service 上都有哪些方法需要执行，参数 `propKeys` 是一个方法名数组，它负责配置 service 上将要执行的函数。

```javascript
function createWebService(baseUrl, propKeys) {
  let service = {};
  propKeys.forEach(function (propKey) {
    Object.defineProperty(service, propKey, {
      get: function () {
        return httpGet(baseUrl+'/'+propKey);
      }
    );
  });
  return service;
}
```

### 3.3 跟踪属性访问
Proxy 可以让我们跟踪指定的属性的读取与修改，为了说明它是如何工作的，我们创建了一个 Point 类并跟踪它的实例的属性。

```javascript
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
  toString() {
    return 'Point('+this.x+','+this.y+')';
  }
}
// 跟踪属性 `x` 和 `y`
let p = new Point(5, 7);
p = tracePropAccess(p, ['x', 'y']);
```

现在，获取和修改 p 的属性会出现下面的输出：

```javascript
> p.x
GET x
5
> p.x = 21
SET x=21
21
```

有趣的是及时父类 Point 访问 p 的属性也会被跟踪，这是因为 this 现在指向的是一个 proxy 了：

```javascript
> p.toString()
GET x
GET y
'Point(21,7)'
```

在 ES5 中如果我们要实现 `tracePropAccess` 就要用 `getter` 和 `setter` 替代每一个属性，它们还需要一个额外的对象 `propData` 用来存属性的数据，需要注意的是我们在完全修改原有的实现，也就是说我们在元编程。

```javascript
function tracePropAccess(obj, propKeys) {
  // 保存属性数据
  let propData = Object.create(null);
  // 使用 getter 和 setter 替代每一个属性
  propKeys.forEach(function (propKey) {
    propData[propKey] = obj[propKey];
    Object.defineProperty(obj, propKey, {
      get: function () {
        console.log('GET '+propKey);
        return propData[propKey];
      },
      set: function (value) {
        console.log('SET '+propKey+'='+value);
        propData[propKey] = value;
      },
    });
  });
  return obj;
}
```

在 ES6 中我们可以用一种更简单的、基于 proxy 的实现，我们拦截属性的 get/set 并且不对原有代码做任何修改。

```javascript
function tracePropAccess(obj, propKeys) {
  let propKeySet = new Set(...propKeys);
  return new Proxy(obj, {
    get(target, propKey, receiver) {
      if (propKeySet.has(propKey)) {
        console.log('GET '+propKey);
      }
      return Reflect.get(target, propKey, receiver);
    },
    set(target, propKey, value, receiver) {
      if (propKeySet.has(propKey)) {
        console.log('SET '+propKey+'='+value);
      }
      return Reflect.set(target, propKey, value, receiver);
    },
  });
}
```

### 3.4 访问不存在的属性时发出提醒
Javascript 对于属性获取非常宽容，举个例子，当你访问一个属性并有意无意拼错了它的名字，Javascript 并不会报错而只是返回一个 `undefined`。在这种场景下你可以用 proxy 以抛出错误，本质上是把 proxy 设置为 Object 的原型。

属性不存在于对象上时，proxy 的 get 就会被触发，如果这个属性也不再原型链中，那么此时我们抛出一个异常。

```javascript
let PropertyChecker = new Proxy({}, {
  get(target, propKey, receiver) {
    if (!(propKey in target)) {
      throw new ReferenceError('Unknown property: '+propKey);
    }
    return Reflect.get(target, propKey, receiver);
  }
});
```

下面我们为我们创建的对象使用 `PropertyChecker` ：

```javascript
> let obj = { __proto__: PropertyChecker, foo: 123 };
> obj.foo  // own
123
> obj.fo
ReferenceError: Unknown property: fo
> obj.toString()  // inherited
'[object Object]'
```

如果我们把 PropertyChecker 转化为一个构造函数，那么我们甚至可以用 ES6 classes 的 extends 来使用它：

```javascript
function PropertyChecker() { }
PropertyChecker.prototype = new Proxy(···);
    
class Point extends PropertyChecker {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}
    
let p = new Point(5, 7);
console.log(p.x); // 5
console.log(p.z); // ReferenceError
```

如果你担心意外创建新的属性，有两个方法可以解决：你可以创建一个 proxy 劫持 set 方法，或者通过 `Object.preventExtensions(obj)` 方法阻止对象的拓展，意思就是 JS 不允许你继续往 obj 上新增属性。

### 3.5 使用负值获取 Array 元素
有些数组方法允许你通过 `-1` 获取数组最后一个元素，倒数第二个通过 `-2` 以此类推。

```javascript
> ['a', 'b', 'c'].slice(-1)
['c']
```

遗憾的是，通过中括号操作符（`[]`）不能这么获取，不过我们可以用 proxy 劫持 get 操作来支持这一特性，下面的 `createArray` 函数创建了一个支付负数索引的数组：

```javascript
function createArray(...elements) {
  let handler = {
    get(target, propKey, receiver) {
      let index = Number(propKey);
      if (index < 0) {
        propKey = String(target.length + index);
      }
      return Reflect.get(target, propKey, receiver);
     }
  };
  // 把数组用 proxy 包裹一层
  let target = [];
  target.push(...elements);
  return new Proxy(target, handler);
}
let arr = createArray('a', 'b', 'c');
console.log(arr[-1]); // c
```

### 3.6 数据绑定
数据绑定的意义主要在于同步对象之间的数据，一个很流行的场景就是视图同模型绑定，当数据改变时视图也会立刻更新。

为了实现数据绑定，我们必须观察对象的变化并对其作出反应，下面的代码中实现了如何观察一个数组：

```javascript
let array = [];
let observedArray = new Proxy(array, {
  set(target, propertyKey, value, receiver) {
    console.log(propertyKey+'='+value);
    target[propertyKey] = value;
  }
});
observedArray.push('a');
```

Output：

```bash
0=a
length=1
```

查看 Addy Osmani’s（译者注：这位老哥很厉害，BackBone 时就走在前面了，最近在搞 PWA）的 [Data-binding Revolutions with Object.observe()](http://www.html5rocks.com/en/tutorials/es7/observe/) 来进一步了解 `Object.observe()`。

### 3.7 销毁引用
销毁引用的原理是这样的：客户端必须通过某个对象（中间件）来访问一个资源，当客户端只用完毕后销毁这个对象，之后对这个对象执行的操作会直接报错。

下面的代码中我们为 resource 创建了一个可销毁的引用，接下来我们试着通过这个引用访问 resource 上的属性是可以的，然后销毁这个引用。当我们再次尝试通过引用访问 resource 时就不行了。

```javascript
let resource = { x: 11, y: 8 };
let {reference, revoke} = createRevocableReference(resource);
    
// 允许访问
console.log(reference.x); // 11
    
revoke();
    
// 禁止访问
console.log(reference.x); // TypeError: Revoked
```

Proxy 非常适合用来做这种劫持并转发的操作，下面我们简单实现了一个 proxy 版本的 `createRevocableReference`：

```javascript
function createRevocableReference(target) {
  let enabled = true;
  return {
    reference: new Proxy(target, {
      get(target, propKey, receiver) {
        if (!enabled) {
          throw new TypeError('Revoked');
        }
        return Reflect.get(target, propKey, receiver);
      },
      has(target, propKey) {
        if (!enabled) {
          throw new TypeError('Revoked');
        }
        return Reflect.has(target, propKey);
      },
      ···
    }),
    revoke() {
      enabled = false;
    },
  };
}
```

我们可以通过之前介绍的 proxy-as-handler 的模式来进一步简化代码，handler 其实就是 Relect 对象，`get` 方法会返回 Reflect 上对应的方法，如果引用被销毁了，那么就抛出一个 TypeError 错误。

```javascript
function createRevocableReference(target) {
  let enabled = true;
  let handler = new Proxy({}, {
    get(dummyTarget, trapName, receiver) {
      if (!enabled) {
        throw new TypeError('Revoked');
      }
      return Reflect[trapName];
    }
  });
  return {
    reference: new Proxy(target, handler),
    revoke() {
      enabled = false;
    },
  };
}
```

不过现在你不需要自己手动销毁引用，ES6 种允许你创建可以销毁的 proxy。

```javascript
function createRevocableReference(target) {
  let handler = {}; // forward everything
  let { proxy, revoke } = Proxy.revocable(target, handler);
  return { reference: proxy, revoke };
}
```

**Membranes**
Membranes 是一种基于销毁引用的概念：执行环境中会运行不安全的代码，代码的沙河环境通过 Membranes 包一层以同系统环境隔离，对象通过两个方向传递 Membrane：

- 代码可能会接收外部的代码
- 代码可能会给外部返回对象

这两种情况都需要销毁引用，返回的对象或方法也需要销毁引用。

不安全的代码执行完毕后，引用都应当被销毁以防止外部继续访问，[Caja Compiler](https://developers.google.com/caja/) 就是一个将第三方 HTML，CSS，JS 安全的插入你的网站的工具，他就是通过 Membrane 实现的。

###  3.8 其他场景
在举几个 proxy 的使用场景：

- 将操作映射到远程对象的本地占位符，比如 web service
- 读写数据库
- 性能监测：拦截方法的执行并记录每个方法的执行时间
- 类型检查：Nicholas Zakas（译者注：就是高级编程的作者啦，不过现在不如本文的作者高产了）用 proxy 实现了 [类型检查](https://www.nczonline.net/blog/2014/04/29/creating-type-safe-properties-with-ecmascript-6-proxies/)

## Proxy API 设计
### 4.1 分层：分离应用层代码和元编程
Firefox 有一个元编程的功能：当你定义一个函数名为 `__noSuchMethod__` 的方法，当你调用不存在的方法的时候就会被调用，下面是一个例子：

```javascript
let obj = {
  __noSuchMethod__: function (name, args) {
    console.log(name+': '+args);
  }
};
// 下面调用的两个方法都不存在,
// 但是我们可以让它表现的好像存在似的
obj.foo(1);    // Output: foo: 1
obj.bar(1, 2); // Output: bar: 1,2
```

这样 `__noSuchMethod__` 就像是 proxy 一样，同 proxy 不同的是，这个方法是对象自身或继承的，它存在的一个问题就是混淆了应用层代码和元编程，应用层的代码可能会无意中创建元编程（译者注：作者应该是指开发者为对象创建一个 `__noSuchMethod` 的应用层方法）

即使在 ES5 中应用层代码和元编程也会时不时的混淆，举个例子，下面几个元编程会出错，因为它们都属于应用层：

- `obj.hasOwnProperty(propKey)`：当原型链上有方法覆盖了内置方法的时候，这个方法可能就报错了，比如 `obj` 是 `{ hasOwnProperty: null }`，推荐的方式是：`Object.prototype.hasOwnProperty.call(obj, propKey)` 或者缩减版：`{}.hasOwnProperty.call(obj, propKey)`。
- `func.call(···)`, `func.apply(···)`：解决方案同上面的方法一样。
- `obj.__proto__`：在大多数 JS 引擎中，__proto__ 都是一个特殊的属性供开发者读写 `obj` 的原型，因此当你把对象用作字典的时候，千万避开把 __proto__ 设置为一个 key。

现在，你应该知道了为什么不应该混淆应用层和元编程，proxy 是有分层的，应用层（proxy 对象）和元编程（handler 对象）是隔离开的。

### 4.2 Virtual objects vs wrappers
Proxy 主要扮演两个角色：
- 作为 wrapper：proxy 包裹了一层目标对象，控制对它们的访问。比如：可销毁的资源和性能分析。
- 作为 virtual objects：它们只是有特殊方法的对象，比如 proxy 将方法调用转发到一个远程的对象上。

### 4.3 虚拟化透明和 handler 封装
Proxy 通过以下两种方式进行屏蔽：
- 使用者无法判断一个对象是不是 proxy
- 使用者无法通过 proxy 获取原对象
这两点赋予了 proxy 冒充另一个对象的能力，如果你需要判断一个对象是不是 proxy 只能自己去实现，下面的 lib.js 代码中暴露了两个函数：其中一个创建了一个 proxy，另一个检查输入的对象是不是 lib.js 创建的 proxy。

```javascript
// lib.js
    
let proxies = new WeakSet();
    
export function createProxy(obj) {
  let handler = {};
  let proxy = new Proxy(obj, handler);
  proxies.add(proxy);
  return proxy;
}
    
export function isProxy(obj) {
  return proxies.has(obj);
}
```

这个模块用了 ES6 的特性 `WeakSet` 来保存 proxy，`WeakSet` 因为不会阻止成员被 GC 所以非常适合这个场景，下面的例子是 lib.js 的使用：

```javascript
// main.js
    
import { createProxy, isProxy } from './lib.js';
    
let p = createProxy({});

console.log(isProxy(p)); // true
console.log(isProxy({})); // false
```
### 4.4  元对象的协议和 Proxy traps
术语「协议」在计算机科学中有很多种含义，其中之一是：

> 协议是通过一个对象完成一系列任务，它包括一组方法和使用这些方法的规则

需要注意的是这个定义同把协议当作借口的定义是不同的（比如：Objective-C），因为这里的协议包含了规则。

ECMAScript 标准定义了如何执行 JS 代码，它里面包含 [如何处理对象的协议](https://tc39.github.io/ecma262/#sec-ordinary-and-exotic-objects-behaviours) ，这个协议主要用在元编程级别，有时也叫 meta object protocol（MOP）。JS MOP 包含了一条：所有的对象都有内置方法，「内置」的含义是这些方法仅仅存在于规范中，JS 引擎有可能有有可能没有，并且 在 JS 中不能直接访问到，内置方法的名字会被放在两个中括号中间。

内置的获取属性方法叫做 `[[GET]]`，假设我们可以用两个中括弧这种语法，那么这个方法在 JS 中的实现大概像下面的代码：

```javascript
// 方法定义
[[Get]](propKey, receiver) {
  let desc = this.[[GetOwnProperty]](propKey);
  if (desc === undefined) {
    let parent = this.[[GetPrototypeOf]]();
    if (parent === null) return undefined;
    return parent.[[Get]](propKey, receiver); // (*)
  }
  if ('value' in desc) {
    return desc.value;
  }
  let getter = desc.get;
  if (getter === undefined) return undefined;
  return getter.[[Call]](receiver, []);
}
```

这段代码中用到的 MOP 方法有以下几个：

- [[GetOwnProperty]] (trap getOwnPropertyDescriptor)
- [[GetPrototypeOf]] (trap getPrototypeOf)
- [[Get]] (trap get)
- [[Call]] (trap apply)

可以看到 `[[Get]]` 方法会去调用其他 MOP 操作，这样的操作叫做衍生操作，不依赖其他操作的叫做基础操作。

**Proxies 中的 MOP**

Proxies 中的 MOP 同一般对象不同，对于一般的对象，衍生操作调用其他操作，而在 Proxy 中，每个操作不是被 handler 劫持就是转发到 target 上。

什么样的操作应当被 proxy 劫持呢？答案之一是只应用于基础操作上，应用在某些衍生操作上也是可以的。在衍生操作使用 Proxy 的一个好处是可以提高性能并且使用上更方便，如果 `get` 上没有 proxy，那么你只能通过 `getOwnPropertyDescriptor` 来实现了，这样做有一个很大的问题就是 `get` 返回的值可能和 `getOwnPropertyDescriptor` 里保存的值并不一样。

**什么样的操作应当被劫持？**

你不能劫持语言中所有的操作，为什么有的操作不能被劫持呢？原因有两个：

首先，稳定的操作是不能被劫持的，一个操作如果总是传入同样的参数总是返回同样的结果，那么它就是稳定的，如果它被 Proxy 劫持就会变得不稳定、不可靠。严格等于（===）就是一个稳定的操作，它不能被 Proxy 劫持。

第二个原因是劫持意味着你可以运行通畅情况下不能执行的自定义代码，这种代码越多，程序越难 debug。

** `get` 对比 `invoke` **

如果你想要通过 ES6 Proxy 创建虚拟方法，你必须通过 `get` 接口返回一个函数，这就带来了一个问题：为什么不创建一个新的方法（比如叫 `invoke`）来帮助我们区分：

- 通过 `obj.prop` 获取属性
- 通过 `obj.prop()` 调用方法

不这么做的原因有两个：

首先，并不是所有的实现都会区分 `get` 和 `invoke`，[Apple 的  JavascriptCore](https://mail.mozilla.org/pipermail/es-discuss/2010-May/011062.html) 就是一个例子。

其次，取出一个方法并且之后用 `call()` 或 `apply()` 调用应当同直接调用这个方法是一样的效果，下面的代码中这两种调用方式应当是等价的，如果我们额外创建了一个 `invoke` 方法，那么就很难维护它们之间相等了。

```javascript
// 直接调用方法
let result = obj.m();
    
// 取出方法并通过 `call` 调用
let m = obj.m;
let result = m.call(obj);
```

**唯一需要 invoke 的可能性**

有的场景如果不区分调用和获取你就无法完成，通过目前的 Proxy API 是无法完成的，举两个例子：auto-binding 和劫持不存在的方法。

首先，通过把对象 obj 的 prototype 设置成一个 proxy 就可以自动绑定方法了：

- 通过 `obj.m` 获取方法 m 的值，返回一个方法，它的 this 指向 obj
- `obj.m()` 进行方法调用

Auto-binding 在方法被用作回调函数的时候很有用，举个例子：

```javascript
let boundMethod = obj.m;
let result = boundMethod();
```

第二点，invoke 可以实现前面提到的 firefox 支持的特性 `__noSuchMethod__`，obj 的原型会被设置成一个 proxy，它会根据访问未知的属性 `foo` 的方式不同而有不同的表现：

- 如果仅仅只是访问 `obj.foo` 那么会返回 `undefined`
- 如果是调用 `obj.foo()` 那么 proxy 就会劫持这个方法返回自定义的内容

### 4.5 强制 Proxy 的不变性
在我们讨论不变性以及如何保证 proxy 的不变性之前，让我们先回顾一下如何通过 non-extensibility 和 non-configurability 来保护对象。

**保护对象**

保护对象有两种方式：

- non-extensibility 保护对象
- non-configurability 保护属性

**Non-extensibility**：如果一个对象是不可拓展的，那么你不能给它添加属性或者修改它的属性：

```javascript
'use strict'; // 开启严格模式以抛出 TypeError 错误
    
let obj = Object.preventExtensions({});
console.log(Object.isExtensible(obj)); // false
obj.foo = 123; // TypeError: object is not extensible
Object.setPrototypeOf(obj, null); // TypeError: object is not extensible
```

** Non-configurability**：一个属性的所有数据都存在 `attributes` 里面，一个属性就好像一条记录，而 `attributes` 是这条记录的字段，举几个例子：

- attribute value 代表属性的值
- attribute writable 是一个布尔值，表示属性的值是否可以修改
- attribute configurable 也是一个布尔值，表示属性的 attribute 是否可以修改

所以如果一个属性是 non-writable 并且 non-configurable，那么它永远是一个只读的属性：

```javascript
'use strict';
    
let obj = {};
Object.defineProperty(obj, 'foo', {
  value: 123,
  writable: false,
  configurable: false
});
console.log(obj.foo); // 123
obj.foo = 'a'; // TypeError: Cannot assign to read only property
    
Object.defineProperty(obj, 'foo', {
  configurable: true
}); // TypeError: Cannot redefine property
```

更多细节（包括 Object.defineProperty() 的原理）可以在「Speaking Javascript 」中的这两个章节看到：

- [Property Attributes and Property Descriptors](http://speakingjs.com/es5/ch17.html#property_attributes)
- [Protecting Objects](http://speakingjs.com/es5/ch17.html#protecting_objects)

**强制不变性**

通常，non-extensibility 和 non-configurability 意味着：

- 通用：它们可以用于任何对象
- 单向：一旦开启就不能改变

Proxy API 通过检查 handler 方法的参数和返回值保证了不变性，下面是几个不变性的例子和 Proxy 是如何强制执行它们（文末会有一个详细的列表）：

- 不变性：Object.isExtensible(obj) 必须返回一个布尔值
- 通过强制要求 handler 返回的值是一个布尔值
- 不变性：Object.getOwnPropertyDescriptor(obj, ···) 必须返回一个对象或者 undefined
- 当 handler 没有返回期望的值时抛出一个 TypeError
- 不变性：如果 Object.preventExtensions(obj) 返回了 `true` 那么之后任何调用必须返回 `false` 并且 obj 必须是 non-extensible 的
- 如果 target 对象不可拓展且 handler 方法返回 true，抛出一个 TypeError
- 不变性：一旦一个对象是 non-extensible 的，Object.isExtensible(obj) 必须总是返回 `false`
- 当 handler 返回的结果同 Object.isExtensible(target) 返回的不等时，抛一个 TypeError 错误

强制不变性有以下几个好处：

- 同其他对象一样，Proxy 也有可拓展性和可配置性，因此得以控制它的不变性，这一点是在允许虚拟化受保护对象下实现的
- 一个受保护的对象不能被 proxy 修改它的本质，这种修改会造成 bug 和恶意的代码

下面的章节举了几个强制不变性的例子

**例子：non-extensible 的原型必须如实返回**

如果目标对象是不可拓展的，`getPrototypeOf` 的劫持必须返回目标对象的原型。

为了证明这一点，我们创建一个 handler 返回一个同目标对象原型不同的值：

```javascript
let fakeProto = {};
let handler = {
  getPrototypeOf(t) {
    return fakeProto;
  }
};
```

当目标对象是可拓展的时候，伪造原型可以成功：

```javascript
let extensibleTarget = {};
let ext = new Proxy(extensibleTarget, handler);
console.log(Object.getPrototypeOf(ext) === fakeProto); // true
```

然而在一个不可拓展的对象上就会报错：

```javascript
let nonExtensibleTarget = {};
Object.preventExtensions(nonExtensibleTarget);
let nonExt = new Proxy(nonExtensibleTarget, handler);
Object.getPrototypeOf(nonExt); // TypeError
```

**例子：不可写和不可配置的对象属性必须如实反映**

如果目标对象有一个既不可写又不可配置的属性，那么在 get 劫持中，handler 必须如实返回这个属性的值。为了证明这一点，我们创建一个为所有属性返回同样的值的 Proxy：

```javascript
let handler = {
  get(target, propKey) {
    return 'abc';
  }
};
let target = Object.defineProperties(
  {}, {
    foo: {
      value: 123,
      writable: true,
      configurable: true
    },
    bar: {
      value: 456,
      writable: false,
      configurable: false
    },
});
let proxy = new Proxy(target, handler);
```

属性 `target.foo`  没有同时配置 non-writable 和 non-configurable，这意味着我们可以返回一个伪造的值：

```bash
> proxy.foo
'abc'
```

然而 `target.bar` 既不可写又不可配置，因此我们可以伪造它的值：

```bash
> proxy.bar
TypeError: Invariant check failed
```

## 5. Proxy API 
本节会快速的介绍一下 proxy API：包括全局对象 Proxy 和 Reflect

### 5.1 创建 proxy

创建一个 Proxy 有两种方式：
- proxy = new Proxy(target, handler)
通过传入目标对象和 handler 创建一个 proxy
- {proxy, revoke} = Proxy.revocable(target, handler)
创建一个可以通过 revoke 函数销毁的 proxy，revoke 可以调用很多次，但是在第一次调用过后任何对 proxy 的操作都会导致 TypeError 的异常

### 5.2 Handler 方法
本节主要介绍 handler 对象可以拦截的方法有哪些和哪些操作会触发拦截器，有的拦截会返回布尔值，`has` 和 `isExtensible` 的拦截器返回的布尔值代表操作的结果，而其他返回布尔值的拦截器表示操作是否成功。

所有对象的拦截器：
- defineProperty(target, propKey, propDesc) → boolean
Object.defineProperty(proxy, propKey, propDesc)
- deleteProperty(target, propKey) → boolean
delete proxy[propKey]
delete proxy.foo // propKey = 'foo'
- enumerate(target) → Iterator
for (x in proxy) ···
- get(target, propKey, receiver) → any
receiver[propKey]
receiver.foo // propKey = 'foo'
- getOwnPropertyDescriptor(target, propKey) → PropDesc|Undefined
Object.getOwnPropertyDescriptor(proxy, propKey)
- getPrototypeOf(target) → Object|Null
Object.getPrototypeOf(proxy)
- has(target, propKey) → boolean
propKey in proxy
- isExtensible(target) → boolean
Object.isExtensible(proxy)
- ownKeys(target) → Array<PropertyKey>
Object.getOwnPropertyPropertyNames(proxy) (只可以用于类型为 String 的键)
Object.getOwnPropertyPropertySymbols(proxy) (只可以用于类型为 Symbol 的键）
Object.keys(proxy) (只可用于可枚举的类型为 String 的键，可枚举性通过 Object.getOwnPropertyDescriptor 检查)
- preventExtensions(target) → boolean
Object.preventExtensions(proxy)
- set(target, propKey, value, receiver) → boolean
receiver[propKey] = value
receiver.foo = value // propKey = 'foo'
- setPrototypeOf(target, proto) → boolean
Object.setPrototypeOf(proxy, proto)

函数的拦截器：

- apply(target, thisArgument, argumentsList) → any
proxy.apply(thisArgument, argumentsList)
proxy.call(thisArgument, ...argumentsList)
proxy(...argumentsList)
- construct(target, argumentsList) → Object
new proxy(..argumentsList)

**基础操作 vs 衍生操作**

下面的几个操作是基础的，它们的功能不依赖其他操作：

- apply
- defineProperty
- deleteProperty
- getOwnPropertyDescriptor
- getPrototypeOf
- isExtensible
- ownKeys
- preventExtensions
- setPrototypeOf

剩下其余的操作都是衍生的，它们可以通过基础操作实现，举个例子：获取对象的属性可以通过使用 `getPrototypeOf` 遍历对象的原型链然后对每个成员调用 `getOwnPropertyDescriptor` 知道找到属性或者达到原型链顶端。

### 5.3 不变性
不变性是对 handler 安全性的约束，本节主要列举 proxy API 的不变性以及它是如何保证的。下面，当你读到「handler 必须这样做」时，这意味着如果它的行为不符合就会抛出一个 TypeError，有的不变性限制返回值，而其他的是限制参数。保证返回值合法的方式有两种：通常，不合法的返回值会抛出一个 TypeError 异常，当需要返回布尔值的时候会强制将不合法的值转化为布尔值。

下面是所有强制不变性的列表（来自：[ECMAScript 6 标准](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-proxy-object-internal-methods-and-internal-slots)）：

- apply(target, thisArgument, argumentsList)
没有不变性的限制
- construct(target, argumentsList)
返回的值必须是对象（不能是 null 或者基础类型）
- defineProperty(target, propKey, propDesc)
如果 target 是不可拓展的，那么 propDesc 不能创建一个 target 没有的属性
如果 propDesc 将属性 configurable 设置为 false，那么 target 必须有一个不可配置的键：propKey
如果 propDesc 尝试去重新定义 target 已有的属性，如果这次操作中违反了 writable 和 configurable 导致了异常，那么 handler 不能返回不可配置的属性是可配置的，而且不能给一个不可配置 + 不可写的属性返回一个与其原始值不同的值。
- getPrototypeOf(target)
返回值必须是对象或者 null
如果目标对象是不可拓展的，handler 必须返回目标对象的原型
- has(target, propKey)
handler 必须隐藏（通过返回不存在）目标不可配置的属性
如果目标对象是不可拓展的，那么它自己的属性是不可被隐藏的
- isExtensible(target)
handler 的返回结果会被强制转化为布尔值
转化为布尔值之后的返回值必须同 `target.isExtensible()` 的结果相等
- ownKeys(target)
handler 必须返回一个对象，这个对象会被当作一个 array-like 的值并且转成数组
返回的每个元素必须是字符串或者 symbol
返回结果必须包括目标对象所有不可配置的属性
如果目标对象是不可拓展的，那么返回结果必须等于目标对象所有的属性（不能有其他的值）
- preventExtensions(target)
handler 的返回结果会被强制转换成布尔值
如果 handler 返回了一个 truthy 的值（意味着操作成功了），那么之后调用 `target.isExtensible()` 必须返回 false
- set(target, propKey, value, receiver)
如果 target 的 propKey 属性是不可写 + 不可配置的，那么返回值必须等于原始值
如果 target 有一个不可配置 + 没有 setter 的属性，那么抛出一个 TypeError（这样的属性不可能写）
- setPrototypeOf(target, proto)
返回结果会被强制转化为布尔值
如果目标对象是不可拓展的，那么原型也不可以改变。这一点是如何保证的？如果目标对象是不可拓展的并且 handler 返回了 truthy 的值（意味着改变成功了）那么 proto 必须同 target 的 prototype 相同，否则抛出一个 TypeError
