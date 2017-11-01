---
title: async/await 的常见问题
date: 2017-06-02 8:42:26
tags: 
  - async
  - javascript
---

> 在 Nodejs 中编写异步代码一直饱受诟病，基于回调函数的原理很容易写出 “箭头型” 代码，有的人甚至因此对 node 产生了阴影。在那之后相继出现 Promise，Generator 和 async/await 试图从语法层面解决这一问题，自 Nodejs 7 中原生支持 async/await 后大部分 nodejs 开发者应该都在用了，本篇文章我们来探讨一些在使用 async/await 时常见的问题

在开始之前先简单介绍一下 async/await 的使用，因为 async/await 是基于 Promise 实现的，我们来看一下以往通过 Promise 的异步流程控制如何用 async/await 重写：

```js
getUser.then(user => {
  return getOrders({ user })
}).then(orders => {});
```

使用 async/await 重写后：

```js
async function() {
  const user = await getUser();
  const orders = await getOrders({ user });
}
```

可以看到 async/await 避免了 Promise 复杂的链式条用，语法上如同编写同步代码，但是实际上结果是异步的。知道这些就足够了，下面我总结了在实际项目中使用 async/await 容易遇到的问题：

1. 不使用 await 调用了一个 async 修饰的函数

```js
async function fetch() {
  return await getUser();
}

var user = fetch(); // <Promise>
```

`async` 函数**总是**会返回一个 Promise，这里 fetch 是一个 async 函数，如果在调用时我们不使用 await 那么结果只会是一个 Promise

2. 不使用 await 调用了一个返回值非 await 修饰的 async 函数

```js
async function fetch() {
  return { name: 'ymy' };
}

var user = fetch(); // <Promise>
```

正如前面的例子所讲，再次强调：`async` 函数**总是**会返回一个 Promise，即使 `fetch` 函数返回的是一个同步的值。

同样的，如果我们返回一个 await 的同步的值，不使用 await 的话结果依然是一个 Promise。

```js
async function fetch() {
  return await { name: 'ymy' };
}

var result = syncReturn(); // <Promise>
```

3. await 一个 async 函数

```js
async function() {
  async function fetch() {
    return await getUser();
  }
  var user = await fetch(); // { name: 'ymy' }
}
```

这种代码虽然结果是正确的，但是是冗余的，如果在 async 函数内没有别的异步调用，直接返回一个 Promise 就足够了：

```js
async function() {
  function fetch() {
    return getUser();
  }
   var user = await fetch(); // { name: 'ymy' }
}
``` 

4. 在循环中使用 async/await

```js
async function() {
  for (let i = 0; i < 10; i++) {
    await sleep(i * 100);
  }
}
```

上面的代码会依次执行 sleep(0)，sleep(100)，sleep(200)…

假设现在我们需要根据一个数组来执行 sleep 函数，下面我们用 forEach 来重写这部分代码：

```js
async function() {
  results.forEach((r, i) => {
    await sleep(i);  // Error
  });
}
```

这段代码在语法检查阶段就会报错，因为 await 所在的函数 scope 内没有 async 关键字，下面我们做一些修改：

```js
async function() { // Scope A
   console.log(1);
  results.forEach(async (r, i) => { // Scope B
    await sleep(i);
  });
  console.log(2);
}
```

这段代码实际的执行顺序是：
- console.log(1)
- console.log(2)
- sleep(0), sleep(100), sleep(200)…

原因是 await 只会阻塞所在 scope 的函数，因此 Scope A 中的代码执行并不会等待 Scope B 中的 async 函数执行完毕，而是运行后立刻运行后面的代码，这个问题有两种解决方案：第一个就是用前面提到的 for 循环，第二种是用 Promise.all 包起来

```js
async function() { // Scope A
   console.log(1);
  await Promise.all(results.map(async (r, i) => { // Scope B
    return await sleep(i);
  }));
  console.log(2);
}
```

![ha](http://oiw32lugp.qnssl.com/2017-06-02-15403364_1194914753923730_1560524376422481920_n%E7%9A%84%E5%89%AF%E6%9C%AC.jpg)

宝宝晚上吃蛤蟆，放个不容易被和谐的图吧（逃
