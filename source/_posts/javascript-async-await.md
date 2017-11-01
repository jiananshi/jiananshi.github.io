---
title: 使用 async/await 在 Javascript 中编写异步代码
date: 2017-01-14 23:20:30
tags:
---

> 在 Javascript 中我们处理异步通常是以回调函数的方式，随着时间的推移人们很快认识到这是一种不好维护的代码风格，基于回调函数的异步代码很容易嵌套并且难以控制流程。Promise 的出现解决了这个问题，之后我们又有了 Generators 来实现协程，本篇文章介绍一下在 ES7 中提出的 async/await 对于异步代码的解决方案。

## 用法
async/await 的使用非常类似 generator 的 */yield，我们先来看一个读取文件的代码 generator 实现版：

```javascript
var fs = require('fs');

function* load() {
	var file = yield fs.readFile('sample.txt');
	return file;
}

load().next().value.then(console.log); // 'sample file contents'
```

使用 async/await 来写：

```javascript
var fs = require('fs'):

async function load() {
	var file = await fs.readFile('sample.txt'):
	return file;
}

load().then(console.log);
```

对于普通的异步函数，我们并不需要 `next()` 这么复杂的流程控制，通常我们只想要返回值，async/await 的语法也更简洁。

## 错误处理
await 后面的表达式通常是 Promise，如果不是的话，会被转成一个 Promise。但是当非 Promise 表达式抛出异常的时候，我们要用 try/catch 处理：

```javascript
async function load() {
	try {
	  var file = await throw(123);
	}  catch(e) {}
}
```

## References
[Async Functions](https://tc39.github.io/ecmascript-asyncawait/)