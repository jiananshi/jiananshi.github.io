---
title: 「译」探索 ECMAScript Decorators
date: 2017-01-07 22:19:38
tags:
---

原文来自 Addy Osmani 发布于 Medium 的：[Exploring ECMAScript Decorators](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.dlq1mmpup)

Iterators，generators 和 array comprehensions；我很激动看到 Javascript 和 Python 越来越像了，今天我们来讨论一下下一个 “很 Python” 的提案 —— Decorators。

（7 月 29 日更新：）Decorators 已经在 TC39 中了，最新的状态可以在[这个 Repo](https://github.com/tc39/proposal-decorators) 里查看，代码示例也[更新](http://tc39.github.io/proposal-decorators/)了。

## Decorator 的语法

Decorators 究竟是什么？在 Python 中，decorators 为调用[高阶函数](https://en.wikipedia.org/wiki/Higher-order_function)提供了简洁的语法，它接收另一个函数作为参数并拓展它的功能，这一切的发生都不需要涉及去修改作为参数的函数。

![decorator code](http://oiw32lugp.qnssl.com/2017-01-08-011543.jpg)

`@` 符号会让编译器知道：我们要使用名字是 `mydecorator` 的 decorator 了，我们的 decorator 接受一个参数(myfunc)并且返回同样的函数，区别在于返回的函数多了一点功能。

Decorators 非常适用于鉴定权限，打日志，限流等等你不希望手动写到逻辑里的功能。

## ES5 和 ES6 中的 Decorators

在 ES5 中实现命令式声明的 decorator 很简单，在 ES2015 中 class 支持拓展，当多个 class 需要共享某个同样的功能时，我们需要一种更好的分发方式。

Yehuda 关于 decorators 的提案为了保持声明式的语法，允许 Decorators 修改 Javascript 的类，属性和对象字面量。

## 在 ES2016 中使用 Decorators 

在 ES2016 中，Decorator 是一个表达式，它返回一个函数并且它的参数有：target，name 和 property descriptor，通过在属性或类上添加一个 `@` 符号开头的表达式来使用它。

### Decorate 一个属性

我们一起来看一个简单的 Cat 类：

![code](http://oiw32lugp.qnssl.com/2017-01-09-013913.jpg)

执行这个类会将函数 `meow` 设置到 `Cat.prototype` 上，类似下面这段代码：

![code](http://oiw32lugp.qnssl.com/2017-01-09-015413.jpg)

假设我们现在想让一个属性或者方法是只读的，我们可以定义一个 `@readonly` 的 decorator：

![code](http://oiw32lugp.qnssl.com/2017-01-09-015535.jpg)

然后将它添加到我们的 meow 属性上：

![code](http://oiw32lugp.qnssl.com/2017-01-09-015613.jpg)

Decorator 只是一个返回函数的表达式，因此 `@readonly` 和 `@something(parameter)` 两种写法都可以。

那么现在回到我们之前的代码，在将 descriptor 设置到 `Cat.prototype` 上之前，JS 引擎首先会执行 decorator：

![code](http://oiw32lugp.qnssl.com/2017-01-09-015928.jpg)

下面我们来验证一下 meow 是否是一个只读的方法：

![code](http://oiw32lugp.qnssl.com/2017-01-09-020023.jpg)

我们很快会讨论到在 class 上使用 decorator，在那之前让我们先看一下 decorator 相关的库，Jay Phelps 的 [https://github.com/jayphelps/core-decorators.js](https://github.com/jayphelps/core-decorators.js) 已经包含了很多实用的 decorators 了。

它通过 import 的方式引入了自己实现的 `@readonly`：

![code](http://oiw32lugp.qnssl.com/2017-01-09-020719.jpg)

它还包含比如 `@deprecate` 等其他 decorators，它会在你调用这个废弃的方法时提醒你：

> 你可以传入一个自定义的 message，还可以传入一个 url 给 hash 参数，最终调用这个方法时会通过 `console.warn` 将它们展示出来

![code](http://oiw32lugp.qnssl.com/2017-01-09-020841.jpg)

### Decorate 一个 Class

根据提案的规范，decorator 接收目标的 constructor 作为参数，这里我们定义一个 `MySuperHero` 的类，我们可以为他定义一个简单的 `@superhero` decorator：

![code](http://oiw32lugp.qnssl.com/2017-01-09-151246.jpg)

我们可以通过使用参数来进一步拓展它：

![code](http://oiw32lugp.qnssl.com/2017-01-09-151354.jpg)

在 ES2016 中，decorators 既可以在类上也可以在属性上使用，它们会自动获得属性名和目标对象，获取 descriptor 可以让 decorator 做很多事，比如使用 getter、在第一次访问属性时做一次 bind 等等。

## ES2016 Decorators 和 Mixins

我从头到尾读了一遍 Reg Braithwaite 的 [ES2016 Decorators as mixins](http://raganwald.com/2015/06/26/decorators-in-es7.html) 和之前发表的 [Functional Mixins](http://raganwald.com/2015/06/17/functional-mixins.html)。Reg 提出了用一个 helper 函数将功能同任何目标混合，这个函数看起来就像下图里的样子：

![code](http://oiw32lugp.qnssl.com/2017-01-09-152723.jpg)

现在我们可以定义一些 mixins 并且用它们去 decorate 别的 Class，现在假设我们有一个 `ComicBookCharacter` 类：

![code](http://oiw32lugp.qnssl.com/2017-01-09-152825.jpg)

目前看起来它没什么卵用，接下来我们用 Reg 的 mixin 来为它定义两个功能：

![code](http://oiw32lugp.qnssl.com/2017-01-09-152921.jpg)

接下来我们可以用 decorator 来 decorate `ComicBookCharacter` 了（注意这里我们使用了两个 decorator）：

![code](http://oiw32lugp.qnssl.com/2017-01-09-153043.jpg)

最后我们来创建一个 `Batman` 的实例：

![code](http://oiw32lugp.qnssl.com/2017-01-09-153118.jpg)

## 通过 Babel 使用 decorators

遗憾的是，Decorators 目前仍处于提案阶段，所以并不能直接编译。谢天谢地，Babel 在某种开发模式中支持编译 Decorators，所以这篇文章中大部分例子都可以直接运行。

`$ babel --optional es7.decorators`

或者可以使用一个 transformer:

![code](http://oiw32lugp.qnssl.com/2017-01-09-153440.jpg)

Babel 还有一个在线的 [online Babel REPL](https://babeljs.io/repl/)，点击 "Experimental" 就可以激活 decorators 功能。

## 额外的实验

我有幸同 Paul Lewis 成为同事，他在[尝试](https://github.com/GoogleChrome/samples/tree/gh-pages/decorators-es7/read-write)应用 decorators 重新规划 DOM 读写的顺序。它的灵感来源于 Wilson Page 的 FastDOM 库，只不过提供了更简洁的 API。Paul 的 decorators 还会在你在 @write 中调用方法或获取属性，或者在 @read 中操作 DOM 的时候发出一个警告。

下面是 Paul 想法的一个例子：

![code](http://oiw32lugp.qnssl.com/2017-01-09-154446.jpg)



