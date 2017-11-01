---
title: Javascript 中的函数绑定
date: 2016-12-31 15:29:38
tags:
---

以前心里一直有一个疑问，Javascript 中 this 既然和声明的位置无关只和调用的方式有关，那么胖箭头定义的函数再次 call/apply 或者 bind 会有影响吗？又或者 bind 过的函数再 new 一下呢？

在 JS 里函数主要有以下几种调用方式：

### 默认绑定

就是最常见的调用一个函数，如果有用 this 那么它指向 global

```javascript
var a = 1;
function a() {
  console.log(this.a); // points to `window.a`
}
```

需要注意的是，在严格模式下不能讲全局对象用于默认绑定，上面的代码会报 `TypeError`

### 显式绑定

用 call 和 apply 指定函数的 this，ES5 中内置了一个方法：`Function.prototype.bind`，用 call/apply 来实现就是：

```javascript
function bind(fn, obj) {
  return function() {
    return fn.apply(obj, arguments);
  }
}
```

### 隐式绑定

引用 Object 上的方法：

```javascript
var obj = {
  a: 1,
  foo: function() {
    console.log(this.a);
  }
};

obj.foo(); // 1
```

但是隐式绑定经常容易遇到的问题就是 **隐式丢失**，比如当我们使用对象方法的引用时

```javascript
var obj = {
  a: 1,
  foo: function() {
    console.log(this.a);
  }
};

var bar = obj.foo();
bar(); // undefined
```

因为 bar 只是对 foo 方法的一个引用，本身是不带任何修饰的，调用时应用了默认绑定。另一种隐式丢失的情况大家应该都遇到过，就是回调：

```javascript
var obj = {
  a: 1,
  foo: function() {
    console.log(this.a);
  }
};
document.querySelector('button').onclick = obj.foo;
```

点击 button 后输出为 `undefined`，跟上面的原因一样，传给 onclick 的只是一个函数的引用。

### new 绑定

要明白 new 操作为什么会绑定 this，先要了解 new 操作在调用构造函数的时候做了哪几件事：

- 创建一个对象
- 将这个对象的 __prototype__ 指到构造函数的 prototype
- 函数调用 this 时绑定的是这个对象

```javascript
function a(num) {
  this.num = num;
  console.log(this.num);
}

new a(10); // 10
```

## 优先级

有了这些基础以后，让我们来研究一下那么文章开篇提的问题：**绑定的优先级**

首先我们比较显式绑定和隐式绑定：

```javascript
function foo() {
  console.log(this.a);
}
var obj = {
  a: 1,
  foo: foo
};
var obj2 = {
  a: 2
};
obj1.foo(); // 1
obj1.foo.call(obj2); // 2
```

由此可以得出，显示绑定的优先级高于隐式绑定。那么隐式绑定和 new 操作哪个优先级更高呢？

```javascript
function foo(a) {
  this.a = a;
}
var obj = {
  foo: foo
};
var bar = new obj.foo(2);
obj.foo(1); 
console.log(obj.a); // 1
console.log(bar.a); // 2
```

由此可见，new 操作比隐式绑定优先级更高，那么最后我们来比较一下 new 操作和显示绑定：

```javascript
function foo(a) {
  this.a = a;
}
var obj = {};
var bar = foo.bind(obj);
bar(1);
var baz = new bar(2);
console.log(obj.a); // 1
console.log(baz.a); // 2
```

最后可以得出：new 操作 > 显式绑定 > 隐式绑定。额外要说的是，ES6 中定义的胖箭头语法是无法被修改的，它的优先级是最高的。