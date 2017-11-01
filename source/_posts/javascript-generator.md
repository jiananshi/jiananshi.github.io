---
title: 「译」深入理解 ES6 Generators
date: 2017-01-08 20:56:25
tags:
---

> 一直想深入学习一下 Generators，这里翻译一篇 2ality 上的文章 [原文链接](http://www.2ality.com/2015/03/es6-generators.html)

本文是 ES6 系列中的一部分：

- [Iterables and iterators in ECMAScript 6](http://www.2ality.com/2015/02/es6-iteration.html)
- [ES6 generators in depth](http://www.2ality.com/2015/03/es6-generators.html)

Generators 是 ECMAScript 6 的新特性，是一个可以被暂停和恢复的函数，它有很多使用场景：iterators，异步编程，等等。这篇文章主要想写一下 generators 的原理以及介绍一下它在现实场景下的应用。

## 1. 总览

Generators 有两个关键的作用：

- 实现 iterables
- 阻塞异步函数调用

### 1.1 通过 generators 实现 iterables

下面的函数通过对象的 properties 返回一个由 [key,value] 组成的 iterable：

```javascript
// `function` 后面的 
// `objectEntries` 是一个 generator
function* objectEntries(obj) {
    let propKeys = Reflect.ownKeys(obj);

    for (let propKey of propKeys) {
        // `yield` 返回一个值并且暂停 generator
        // 下一次执行函数会从上一次暂停的地方继续
        yield [propKey, obj[propKey]];
    }
}
```

我们稍后会详细介绍 `objectEntries` 的原理，现在让我们先来看一下它的用法：

```javascript
let jane = { first: 'Jane', last: 'Doe' };
for (let [key,value] of objectEntries(jane)) {
   console.log(`${key}: ${value}`);
}
// Output:
// first: Jane
// last: Doe
```

### 1.2 阻塞异步调用

下面的代码中我使用 [co](https://github.com/tj/co) 来异步读取两个 JSON 文件，函数的执行会在 A 行阻塞直到 `Promise.all()` 返回，这意味着我们可以像写同步函数一样写异步函数。

```javascript
co(function* () {
    try {
        let [croftStr, bondStr] = yield Promise.all([  // (A)
            getFile('http://localhost:8000/croft.json'),
            getFile('http://localhost:8000/bond.json'),
        ]);
        let croftJson = JSON.parse(croftStr);
        let bondJson = JSON.parse(bondStr);
    
        console.log(croftJson);
        console.log(bondJson);
    } catch (e) {
        console.log('Failure to read: ' + e);
    }
});
```

`getFile(url)` 返回 url 指向的文件，稍后我们会讲解它的内部实现以及 co 的原理。

## 2. Generators 是什么？
Generators 是一种可以被暂停和恢复的函数，这一特性让它有了很多应用场景。

先来看一个例子：

```javascript
function* genFunc() {
    console.log('First');
    yield; // (A)
    console.log('Second'); // (B)
}
```

`genFunc` 和普通的函数声明有两个最主要的区别：

- 它以 `function*` 开头
- 由于 `yield`，它在执行过程中暂停了

调用 `genFunc` 并不会执行它，而是返回一个调用过的 generator 对象供我们控制 `genFunc` 的执行。

`> let genObj = genFunc();`

`genFunc()` 一开始会在函数体的开头挂起，`genObj.next()` 会继续执行 `genFunc()` 直到遇到下一个 yield 声明。

```javascript
> genObj.next()
First
{ value: undefined, done: false }
```

可以看到 `genObj.next()` 返回了一个对象，现在让我们暂时忽略它，等到了将 generators 用作 iterators 的时候再详细讲它。

`genFunc` 现在在 A 行挂起了，如果我们再次调用 `next()` ，函数会继续执行到 B 行。

```javascript
> genObj.next()
Second
{ value: undefined, done: true }
```

现在这个函数已经执行完毕了，再次调用 `next()` 也不会有任何作用了。

### 2.1 创建 Generators 的方法
你有四种创建 generators 的方法：
1. Generator 函数声明：
```javascript
function* genFunc() { ··· }
let genObj = genFunc();
```
2. Generator 函数表达式：
```javascript
const genFunc = function* () { ··· };
let genObj = genFunc();
```
3. Generator 对象字面量：
```javascript
let obj = {
    * generatorMethod() {
        ···
    }
};
let genObj = obj.generatorMethod();
```
4. 在类定义中的方法（可以是类的定义，也可以是类表达式）：
```javascript
class MyClass {
    * generatorMethod() {
        ···
    }
}
let myInst = new MyClass();
let genObj = myInst.generatorMethod();
```

### 2.2 Generators 的使用场景

Generators 有三种使用场景：

1. Iterators（生产者）：每一个 yield 都可以通过 `next()` 返回一个值，也就是说 generators 可以通过循环和递归创建一组连续的值，由于 generator 对象实现了 Iterable 接口，任何支持 iterables 的 ECMAScript 6 结构都可以消费这些数据，举两个例子：for-of 循环和 spread 操作符（`...`）。
2. Observers（消费者）：yield 还可以通过 next 接收参数，generators 可以作为消费者在调用 next 的时候不断地消费数据。
3. Coroutines（生产者和消费者）：既然 generators 可以挂起，并且可以同时作为消费者和生产者使用，我们很容易可以将它转化为 coroutines。

下面我们详细介绍一下这三个场景。

## 3. Generators 用作 iterators（生产者）
这一节我们主要来介绍将 generator 用作生产者，也就是说 generator 函数的返回既是一个 Iterable 也是一个 Iterator。

```javascript
interface Iterable {
    [Symbol.iterator]() : Iterator;
}
interface Iterator {
    next() : IteratorResult;
    return?(value? : any) : IteratorResult;
}
interface IteratorResult {
    value : any;
    done : boolean;
}
```

Generator 函数通过 yield 创建了一组值，消费者调用 iterator 的 `next()` 方法不断的消费这个值，举个例子，下面的 generator 创造了一组值：’a’ 和 ‘b’：

```javascript
function* genFunc() {
    yield 'a';
    yield 'b';
}
```

下面的代码展示了如何通过 generator 对象 `genObj` 来获取 yield 后的值：

```javascript
> let genObj = genFunc();
> genObj.next()
{ value: 'a', done: false }
> genObj.next()
{ value: 'b', done: false }
> genObj.next() // 循环结束
( value: undefined, done: true }
```

### 3.1 如何遍历 generator

既然 generator 是 iterable 的，ES6 中支持 iterables 的数据结构都可以应用在它下身上，这其中有三个值得我们注意。

首先是 for-of 循环：

``` javascript
for (let x of genFunc()) {
    console.log(x);
}
// Output:
// a
// b
```

第二个是 spread 操作符（`...`），它将一组数据转化为数组：

`let arr = [...genFunc()]; // ['a', 'b']`

最后是解构（destructuring）：

```javascript
> let [x, y] = genFunc();
> x
'a'
> y
'b'
```

### 3.2 Generator 的返回值

前面的 generator 函数没有 return，也就是返回值是 undefined，这里我们尝试显式的让 generator 返回一个值：

```javascript
function* genFuncWithReturn() {
    yield 'a';
    yield 'b';
    return 'result';
}
```

返回值在 `next()` 最后一个返回，done 为 true 的对象里：

```javascript
> let genObjWithReturn = genFuncWithReturn();
> genObjWithReturn.next()
{ value: 'a', done: false }
> genObjWithReturn.next()
{ value: 'b', done: false }
> genObjWithReturn.next()
{ value: 'result', done: true }
```

然而大部分支持 iterables 的数据结构都没有输出 done 为 true 的对象里的值：

```javascript
for (let x of genFuncWithReturn()) {
    console.log(x);
}
// Output:
// a
// b
    
let arr = [...genFuncWithReturn()]; // ['a', 'b']
```

我们稍后会介绍递归操作符 yield*，它会考虑 done 为 true 的对象的值。

### 3.3 示例：遍历属性

接下来我们会写个例子展示用 generators 实现 iterables 有多么方便，下面代码里的 `objectEntries()` 函数返回一个对象所有属性的的 iterable：

```javascript
function* objectEntries(obj) {
    // 在 ES6 中你既可以用字符串，也可以用 Symbol 
    // 作为属性的键(key)
    // 使用 Reflect.ownKeys() 则两者都会返回
    let propKeys = Reflect.ownKeys(obj);
    
    for (let propKey of propKeys) {
        yield [propKey, obj[propKey]];
    }
}
```

上面的函数让你可以通过 for-of 遍历对象 jane：

```javascript
let jane = { first: 'Jane', last: 'Doe' };
for (let [key,value] of objectEntries(jane)) {
    console.log(`${key}: ${value}`);
}
// Output:
// first: Jane
// last: Doe
```

让我们来对比一下不用 generators 实现一个同样功能的 `objectEntries()` 有多么复杂：

```javascript
function objectEntries(obj) {
    let index = 0;
    let propKeys = Reflect.ownKeys(obj);
    
    return {
        [Symbol.iterator]() {
            return this;
        },
        next() {
            if (index < propKeys.length) {
                let key = propKeys[index];
                index++;
                return { value: [key, obj[key]] };
            } else {
                return { done: true };
            }
        }
    };
}
```

### 3.4 通过 yield* 实现递归

yield* 操作符可以让你在 generator 里调用另一个 generator，接下来我只会讲解两个 generator 都有输出的情况，稍后我会讲解如果同时有 Input 参与的情况。

一个 generator 如何递归的调用另一个 generator？现在假设你写了一个 generator 叫做 foo：

```javascript
function* foo() {
    yield 'a';
    yield 'b';
}
```

怎么样才能在 generator bar 中调用 foo？（注意下面的方式是没用的）：

```javascript
function* bar() {
    yield 'x';
    foo(); // 什么也没有做！
    yield 'y';
}
```

调用 `foo()` 会返回一个对象，但是并不会真正执行 `foo()`，因此 ECMAScript 6 定义了 `yield*` 操作符用来递归调用 generator：

```javascript
function* bar() {
    yield 'x';
    yield* foo();
    yield 'y';
}
    
// 将所有 bar 返回的 yield 后的值放到一个数组里
let arr = [...bar()];
    // ['x', 'a', 'b', 'y']
```

yield* 的作用类似下面的代码：

```javascript
function* bar() {
    yield 'x';
    for (let value of foo()) {
        yield value;
    }
    yield 'y';
}
```

yield* 后面的操作符不一定要求非得是 generator，可以是任何 iterable：

```javascript
function* bla() {
    yield 'sequence';
    yield* ['of', 'yielded'];
    yield 'values';
}
    
let arr = [...bla()];
    // ['sequence', 'of', 'yielded', 'values']
```

yield* 会处理 iteration 结束时的值

大部分支持 iterables 的数据结构会在 iteration 结束时（done 为 true）忽略最后的值，Generators 通过 return 提供这个值：

```javascript
function* genFuncWithReturn() {
    yield 'a';
    yield 'b';
    return 'The result';
}
function* logReturned(genObj) {
    let result = yield* genObj;
    console.log(result); // (A)
}
```

下面我们通过 `logReturned()` 输出所有值：

```javascript
> [...logReturned(genFuncWithReturn())]
The result
[ 'a', 'b' ]
```

**遍历树**

通过递归遍历一棵树很容易，但是写一个 iterator 遍历树就很复杂了，使用 generators 可以让你很容易通过递归实现一个 iterator。举个例子，下面的数据结构是一颗二叉树，因为它有一个 Symbol.iterator 方法，所以它是 iterable 的，而且这个方法是个 generator，可以通过调用它获得一个 iterator。

```javascript
class BinaryTree {
    constructor(value, left=null, right=null) {
        this.value = value;
        this.left = left;
        this.right = right;
    }
    
    /** Prefix iteration */
    * [Symbol.iterator]() {
        yield this.value;
        if (this.left) {
            yield* this.left;
        }
        if (this.right) {
            yield* this.right;
        }
    }
}
```

下面的代码创建了一个二叉树并且通过 for-of 遍历它：

```javascript
let tree = new BinaryTree('a',
    new BinaryTree('b',
        new BinaryTree('c'),
        new BinaryTree('d')),
    new BinaryTree('e'));
    
for (let x of tree) {
    console.log(x);
}
// Output:
// a
// b
// c
// d
// e
```

### 3.5 你只能在 generators 中使用 yield

Generators 一个最明显的限制就是你只能在 generator 函数里使用 yield，也就是说回调函数里的 yield 并不会起作用：

```javascript
function* genFunc() {
    ['a', 'b'].forEach(x => yield x); // SyntaxError
}
```

yield 不允许出现在一个非 generator 函数中，所以上面的代码报错了。上面这种情况我们可以把它改成非 callback 的模式，但是并不是所有场景都能这么做。

```javascript
function* genFunc() {
    for (let x of ['a', 'b']) {
        yield x; // OK
    }
}
```

## 4. Generators 作为 observers（消费者）
```javascript
interface Observer {
    next(value? : any) : void;
    return(value? : any) : void;
    throw(error) : void;
}
```

Generator 作为一个 Observer，在它接收到输入之前都处于暂停状态，输入有三种类型：

- `next()` 发送普通输入
- `return()` 中止 generator
- `throw()` 发出一个 error 信号

### 4.1 通过 next 发送数据

如果你把 generator 当作 observer 用，你通过 `next()` 向它发送数据，它通过 `yield` 来接收：

```javascript
function* dataConsumer() {
    console.log('Started');
    console.log(`1. ${yield}`); // (A)
    console.log(`2. ${yield}`);
    return 'result';
}
```

首先，我们创建一个 generator 对象：

`> let genObj = dataConsumer();`

现在我们开始调用 `genObj.next()` 来启动 generator，函数一直执行到第一个 yield 声明，`next()` 函数的结果就是 A 行（undefined）。我们现在并不关心 `next()` 的返回，因为目前我们只是在用它发送值而不是接收。

```javascript
> genObj.next()
Started
{ value: undefined, done: false }
```

现在我们再调用两次 `next()`，一次传值 ‘a’ 一次传 ‘b’：

```javascript
> genObj.next('a')
1. a
{ value: undefined, done: false }
    
> genObj.next('b')
2. b
{ value: 'result', done: true }
```

最后一个 `next()` 的值就是 `dataConsumer()` 的返回值，done 属性为 true 意味着 generator 执行完毕了。

**首次调用 next()**

当我们在把 generator 作为 observer 用的时候，时刻记住：`next()` 的第一次执行仅仅是为了启动 observer。之后它才可以接收输入，因为首次调用要比 yield 先执行，因此你不能在第一次调用 `next()` 的时候传参数：

```javascript
> function* g() { yield }
> g().next('hello')
TypeError: attempt to send 'hello' to newborn generator
```

下面的 utility 函数解决了这个问题：

```javascript
/**
 * 返回一个函数
 * 调用这个函数的时候返回调用过 `next()` 的 generator
 */
function coroutine(generatorFunction) {
    return function (...args) {
        let generatorObject = generatorFunction(...args);
        generatorObject.next();
        return generatorObject;
    };
}
```

下面我们来对比一下 `coroutine()` 和一个普通的 `generator`

```javascript
const wrapped = coroutine(function* () {
    console.log(`First input: ${yield}`);
    return 'DONE';
});
const normal = function* () {
    console.log(`First input: ${yield}`);
    return 'DONE';
};
```

Generator `wrapped` 可以立刻接收输入：

```javascript
> wrapped().next('hello!')
First input: hello!
```

Generator `normal` 需要执行一次额外的 `next()` 才能接收输入：

```javascript
> let genObj = normal();
> genObj.next()
{ value: undefined, done: false }
> genObj.next('hello!')
First input: hello!
{ value: 'DONE', done: true }
```

### 4.2 yield 绑定很松散

yield 绑定十分松散，所以我们不需要把操作符放到括号里：

`yield a + b + c;`

等价于：

`yield (a + b + c);`

而不是：

`(yield a) + b + c;`

很多操作符比 yield 绑定更紧凑，当你把 yield 作为操作符使用时，你需要把它放在括号中，举个例子，下面的代码没有把 yield 放在括号里所以导致了 SyntaxError：

```javascript
console.log('Hello' + yield); // SyntaxError
console.log('Hello' + yield 123); // SyntaxError
    
console.log('Hello' + (yield)); // OK
console.log('Hello' + (yield 123)); // OK
```

当 yield 是参数时不需要括号：

`foo(yield 'a', yield ‘b’);`

当 yield 是一个右值时也不需要括号：

`let input = yield;`
