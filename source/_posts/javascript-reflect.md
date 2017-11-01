---
title: 「译」Reflect in Javascript
date: 2017-01-15 23:59:14
tags:
---

> 在 harmony-reflect 项目中看到一篇介绍 Reflect 的文章，翻译一篇到这里 [原文链接](https://github.com/tvcutsem/harmony-reflect/wiki#reflect)

Reflect 对象提供了很多实用的函数，其中有很多和 ES5 全局 Object 的方法重合了（具体是哪些可以查看 [harmony-reflect/api.md](https://github.com/tvcutsem/harmony-reflect/blob/master/doc/api.md) ），下面我想列举一下你为什么应该现在就开始使用 Reflect 的几个原因：

Reflect 有很多跟 ES5 Object 上的方法类似的方法，比如 `Reflect.getOwnPropertyDescriptor` 和 `Reflect.defineProperty`，不过 `Object.defineProperty(obj, name, desc)` 会在属性成功定义后返回 `obj` 失败的时候抛出一个 `TypeError`，而 `Reflect.defineProperty(obj, name, desc)` 只会返回一个布尔值表示属性是否成功定义，下面是两者的代码示例：

```javascript
try {
  Object.defineProperty(obj, name, desc);
  // 属性定义成功
} catch (e) {
  // 属性定义有可能出错
}
```

用 Reflect：

```javascript
if (Reflect.defineProperty(obj, name, desc)) {
  // 成功
} else {
  // 失败
}
```

其他的方法比如：`Reflect.set`（更新一个属性）、`Reflect.deleteProperty`（删除属性）、`Reflect.preventExtensions`（禁止拓展对象）、`Reflect.setPrototypeOf`（更新对象的原型链）也都是通过布尔值表示操作结果。

## 函数式操作
在 ES5 中检查 `obj` 上是否定义或者继承了 `name` 属性通过 `(name in obj)` 来完成，删除一个属性通过 `delete obj[name]` ，虽然语法非常简练但是同时你在将它们作为参数使用时用函数包裹起来。

有了 Reflect 这些操作可以以函数的方式使用：`Reflect.has(obj, name)` 等价于 `(name in obj`，而 `Reflect.deleteProperty(obj, name)` 等价于 `delete obj[name]`。

## 更加健壮的函数
在 ES5 中实现将函数 `f` 的 this 引用指向 `obj` 并且传些参数 `args` 代码如下：

`f.apply(obj, args)`

`f` 可能有意无意的实现了自己的 `apply` 方法，为了确保使用的是内置的 `apply` 通常人们会这么写：

`Function.prototype.apply.call(f, obj, args)`

这种写法不仅啰嗦而且很难懂，通过 Reflect 我们可以简洁高效的重写上面的代码：

`Reflect.apply(f, obj, args)`

## 不定参数的构造函数
在 ES6 中可以通过 spread 语法来使用未知数量的参数：

`var obj = new F(...args)`

在 ES5 种由于我们只能用 `F.call` 或者 `F.apply` 来传递不定长度的参数，但是这样我们无法去 `new F` 了，用 Reflect 可以这样写：

`var obj = Reflect.construct(F, args)`

## 控制访问器的 this 绑定
在 ES5 中访问或更新属性很简单：

```javascript
var name = ... // 获取字符串属性名
obj[name] // 寻找属性值
obj[name] = value // 更新属性
```

`Reflect.get` 和 `Reflect.set` 可以完成同样的事，区别是它额外接收一个 `receiver` 参数允许你显式的定义 this 绑定：

```javascript
var name = ... // 获取字符串属性名
Reflect.get(obj, name, wrapper) // 如果 obj[name] 是访问器, this 会被设置为 `wrapper`
Reflect.set(obj, name, value, wrapper)
```

当你定义了一个 `obj` 并且希望任何访问自身属性的访问器重定向到 `wrapper` 的时候：

```javascript
var obj = {
  get foo() { return this.bar(); },
  bar: function() { ... }
}
```

调用 `Reflect.get(obj, "foo", wrapper)` 会使 `this.bar()` 重定向到 `wrapper`。

## 避免直接访问 __proto__
在有的浏览器里你可以通过 `__proto__` 直接访问对象的原型，ES5 标准有一个方法: `Object.getPrototypeOf(obj)` 用来获取原型，`Reflect.getPrototypeOf(obj)` 的功能一模一样，区别在于它还有一个 `Reflect.setPrototypeOf(obj, newProto)` 方法用来设置对象的原型，这是新的兼容 ES6 的更新对象原型的方法。
