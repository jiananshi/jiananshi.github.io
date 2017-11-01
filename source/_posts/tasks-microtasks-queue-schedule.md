---
title: 「译」Tasks, microtasks, 队列和执行顺序
date: 2017-03-18 03:06:28
tags: eventloop javascript microtasks 
---

> 搁了好久没更博客，再不写要被某人 BS 了，挑了篇之前看过的关于 microtasks 很好的文章（来自 Google Chrome 的开发），看完会对 setTimeout/process.nextTick/Mutation.Observer 等等 task 和 microtasks 的概念更清晰些，当然如果这些名词对你来说感到陌生也不妨碍阅读。

当我告诉我的同事 [Matt Gaunt](https://twitter.com/gauntface) 我在考虑写一篇关于 microtask  队列和浏览器 event loop 的执行相关的文章时，他对我说：「说真的 Jake，我一定不会去读这篇文章的」，不过我还是写了，现在让我们坐下来一起阅读它吧，好吗？

如果你更偏向于通过视频的方式学习， [Philip Roberts](https://twitter.com/philip_roberts) 在 JSConf 上有一次 [关于 event loop 精彩的演讲](https://www.youtube.com/watch?v=8aGhZQkoFbQ) - 虽然只字未提 microtasks，但那仍然是个不错的补充。

接下来我们来看一段 JavaScript 代码：

```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

log 的执行顺序会是什么呢？

正确的答案是：`script start` => `script end` => `promise1` => `promise2` => `setTimeout`，但是结果也会根据浏览器的支持而有所不同。

Microsoft Edge, Firefox 40, iOS Safari and desktop Safari 8.0.8 在 `promise1`和 `promise2` 之前打印了 `setTimeout` - 虽然这可能是一个竞态条件，但这仍然很不科学，Firefox 39 and Safari 8.0.7 总是能输出正确的结果。

## 为什么会这样呢
为了理解上面的代码，你需要知道 event loop 是如何处理 taks 和 microtasks 的，当你第一次听到这个名词的时候你的脑袋可能会一片空白，深呼吸...然后让我们继续往下看

每个「线程」会有它自己的 **event loop**，所以每个 web worker 都会有一个独立的 event loop，因此它们可以独立运行互不干扰，在同一域名下所有的窗口共享一个 event loop，因此它们可以进行同步的通信。event loop 会不间断的运行，执行队列里的任务。一个 event loop 的任务可能有多个来源，这样的设计保证了每个任务源的任务在执行时的顺序（有的规范会自定义这个顺序，比如 [IndexedDB](http://w3c.github.io/IndexedDB/#database-access-task-source)），浏览器需要在每次事件循环中决定执行哪一个任务源，这样的话浏览器可以将高优先级的任务排在前面，比如响应用户输入。

**Tasks** 是有计划执行的，这意味着浏览器可以保证 Javascript/DOM 的执行顺序，浏览器有可能在每个 task 的间隔更新视图。一个点击事件的回调函数需要创建一个 task，渲染 HTML 也是，还有上面例子中的 `setTimeout` 也需要创建一个 task。

`setTimeout` 在间隔一定时间后为它的回调函数创建一个 task，所以上面例子中 `setTimeout` 的输出在 `script end` 之后，因为打印 `script end` 也是第一个任务的一部分工作，`setTimeout` 在另一个 task 中。现在我们快要搞定这个概念了，不过我需要你集中注意力看接下来的话题。

**Microtasks** 通常被用在当前执行脚本结束后需要立刻执行的任务，比如对一系列操作做出响应，或者完成一些异步任务而无需新建一个 task。每个任务结束时，任何 Javascript 代码都执行完毕后，浏览器才会处理 microtasks 队列，microtasks 执行中创建的 microtasks 队列都会被放到本次执行队列的末端被执行。Microtasks 包括 mutation observer 回调函数，以及上面例子中的 Promise 回调。

当一个 Promise 执行完毕后，它会为后面的回调（.then）创建新的 microtasks，这意味着即使 Promise 已经执行完毕，后续的 `then` 方法调用仍然是异步的，在一个执行完毕的 Promise 上调用 `.then(yey, any)` 会立刻创建一个 microtasks 队列。因此在上面的例子中 `promise1` 和 `promise2` 在 `script end` 之后打印，因为 microtasks 必须等任务中的 Javascript 都执行完毕后才会开始执行，而它比 `setTimeout` 早执行是因为 microtasks 总是在下一个任务之前执行。

下面来看一个例子（译者注：博主写了一个交互式查看代码运行时序的 DEMO，推荐去原文地址点一点看一下： [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) ）

### 为什么有的浏览器表现不是这样的
有的浏览器运行文章开头的例子会输出：`script start`，`script end`，`setTimeout`，`promise1`，`promise2`。它们在 `setTimeout` 之后输出 Promise 的回调，看起来就像是它们为 Promise 回调创建了一个新的 task 而不是 microtasks。

某个方面上来说，这是可以理解的，Promise 的概念来自于 ECMAScript 而不是 HTML 规范，ECMAScript 有一个 “jobs” 的概念，类似于 microtasks，但是它们之间的关系并不明确（可以参考这个 [邮件讨论](https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16)）。不过不管怎样，Promise 应该是 microtasks 是一个共识。

将 Promise 当作 tasks 执行会导致性能问题，回调函数有可能会不必要的被 task 相关的任务延迟（比如界面渲染），由于同其他任务源的竞争，这也会带来一定的不确定性，同时还可能会破坏同其他 API 的交互，后面我们会详细的讨论这一点。

Promise microtasks 支持的状态可以在 [这里](https://connect.microsoft.com/IE/feedback/details/1658365) 看到，WebKit nightly 的处理是正确的，所以我们可以假设最终 Safari 的表现也会正常，而 Firefox 43 修复了这个问题。

值得一提的是 Safari 和 Firefox 都在这个问题上被修复之前有过反复，我个人猜测这只是个巧克。

## 如何分辨一个任务是 tasks 还是 microtasks 
有一个方法是通过打印 log，同 Promise 和 setTimeout 对比，不过这一点依赖于 Promise/setTimeout 的正确实现。

推荐的方式是去阅读标准，举个例子： [step 14 of setTimeout](https://html.spec.whatwg.org/multipage/webappapis.html#timer-initialisation-steps) 创建了一个 tasks 队列，[step 5 of queuing a mutation record](https://dom.spec.whatwg.org/#queue-a-mutation-record) 创建了 microtasks 队列。

正如我前面提到的，ECMAScript 中把 microtasks 叫做 “jobs”，在 [step 8.a of PerformPromiseThen](http://www.ecma-international.org/ecma-262/6.0/#sec-performpromisethen) 中，`EnqueueJob` 被称作把 microtask 放入队列。

现在我们来看一个更复杂的例子。

## 一级关卡
在撰写本文之前我在这点上搞错了，下面是部分 html 代码：

```html
<div class="outer">
  <div class="inner"></div>
</div>
```

运行下面的 JS 代码，当我点击 `div.inner` 的时候，输出应该是什么？

```javascript
// Let's get hold of those elements
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// …which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

### 测试

点击内部的元素试试看（译者注：作者写了个 demo 展示上面的代码，并且在文章中做了交互，推荐去原文观看 [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) ）

输出是：

```bash
click
promise
mutate
click
promise
mutate
timeout
timeout
```

跟你预想的结果一样吗？你的设想可能正确的，不过遗憾的是浏览器对此的表现不一致：

![browser](http://oiw32lugp.qnssl.com/2017-03-17-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-18%20%E4%B8%8A%E5%8D%882.16.31.png)

### 谁是正确的？

`click` 事件一定是一个 task，Mutation observer 和 Promise 回调被放入 microtasks 队列，`setTimeout` 是一个 task。因此代码应该是这样运行的：

（译者注：同上，作者放了一个有步骤的交互和 log 输出，推荐原文查看）

在 Chrome 中的表现是正确的，对我来说 “意想不到” 的地方是 microtasks 在回调函数之后立刻就执行了（虽然此时没有任何 Javascript 在执行），我过去以为它的执行必须在任务结束前，这条规则来自于 HTML 标准里关于回调函数的部分：

> 如果 [堆栈是干净的](https://html.spec.whatwg.org/multipage/webappapis.html#stack-of-script-settings-objects) ，进行 microtask 的检查
> —— [HTML: Cleaning up after a callback](https://html.spec.whatwg.org/multipage/webappapis.html#clean-up-after-running-a-callback) 第三步

… microtask 的检查包含遍历 microtasks 队列，除非我们正在处理 microtasks 队列。类似的，ECMAScript 是这样描述 jobs 的：

> Job 的执行可以在当前没有执行环境并且执行环境的堆栈是干净的情况下进行
> —— [ECMAScript: Jobs and Job Queues](http://www.ecma-international.org/ecma-262/6.0/#sec-jobs-and-job-queues)

… 虽然 “可以” 的描述在 HTML 里变成了 “必须”。

（译者注：这部分不看例子的话会比较难理解，我也看了很多次，作者是指在例子中 `click` 事件从内部 dom 冒泡到外部，Javascript 回调的 task 一直在执行，作者理解 microtasks 队列的处理应该被限制在任务结束时执行，也就是外层 dom 的回调结束后，然而事实是当前执行环境中堆栈是干净的就会开始处理 microtasks 队列）

### 浏览器哪里做错了？
Firefox 和 Safari 对于 microtasks 队列中的 Mutation 回调的处理是正确的，而 promises 回调的处理是不同的，这种情况可以理解，毕竟 jobs 和 microtasks 的关系十分模糊，但我仍希望它们可以在事件回调后被处理。

在最新的版本中 Promise 回调已经可以被正确的处理了，但是在点击事件回调间隔函数中对 microtasks 队列的处理依然是有问题的，它在所有事件回调结束之后才运行 microtasks 队列，结果就是 `mutation` log 只输出了一次。

## 一级关卡增强版
继续上面的例子，如果我们执行下面的代码会怎样？

```javascript
inner.click();
```

这段代码运行后仍然会像上面的事件流一样，不同的是这里是脚本而不是真实的交互。

浏览器的输出如下：

![browser](http://oiw32lugp.qnssl.com/2017-03-17-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-18%20%E4%B8%8A%E5%8D%882.47.05.png)

我发誓在 Chrome 中我总是拿到不同的结果，一开始我还以为是因为我用的 Canary 版本，如果你在 Chrome 中运行的结果和图里的不同，请在留言中告诉我你的 Chrome 版本。

### 为什么输出会不同？
下面是代码运行的例子（译者注：同上，推荐去原文看，这里坐着用图例 + 步骤很好的展示了堆栈的情况，一看就理解上面代码的输出的原因了）。

所以正确的顺序应该是：`click`, `click`, `promise`, `mutate`, `promise`, `timeout`, `timeout`，Chrome 又对了一次。

还记得前面提到的吗？在每个事件回调调用后：

> 如果 [堆栈是干净的](https://html.spec.whatwg.org/multipage/webappapis.html#stack-of-script-settings-objects) ，进行 microtask 的检查
> —— [HTML: Cleaning up after a callback](https://html.spec.whatwg.org/multipage/webappapis.html#clean-up-after-running-a-callback) 第三步

在前面的例子中这意味着 microtasks 在事件回调中被处理，而这里 `.click()` 导致事件的分发是同步的，所以运行 `.click()` 的脚本在回调中依然存在堆栈中，上面提到的规则确保 microtasks 不会在 Javascript 运行过程中处理，这意味着只有当所有的事件回调都执行完后才会处理 microtasks。

## 知道这些有什么用？
这个是一个模糊的知识点，我在尝试 [使用 Promise 为 IndexedDB 创建一个容器库](https://github.com/jakearchibald/idb/blob/master/lib/idb.js) 的时候遇到了这个问题。

当 IDB 发出一个成功的事件后，相关的对象应当 [在事件分发后注销 step 4](http://w3c.github.io/IndexedDB/#fire-a-success-event) ，如果我在成功事件后 resolve 掉一个 Promise，回调函数会在 step 4 之前运行，这时相关对象仍然是活跃的。

你可以在 Firefox 里解决这个问题，因为类似 [es6-promise](https://github.com/jakearchibald/es6-promise) 这样的库正确的使用了 microtasks，用 mutation observers 作为回调。Safari 在这类问题上存在严重的竞态条件，但这可能只是它们对于 [IDB 的实现有问题](http://www.raymondcamden.com/2014/09/25/IndexedDB-on-iOS-8-Broken-Bad)，遗憾的是在 IE/Edge 中总是挂的，mutation 事件压根没有在回调后被处理。

希望未来浏览器的表现能更加一致。

## 你成功了！
总结：
- Tasks 是按顺序执行的，浏览器可能在它们的间隔渲染视图
- Microtasks 也是按顺序失踪的，并且在下面两种情况下执行：
   1. 在每个没有 JavaScript 运行的回调后
   2. 在每个 task 结束时
  
希望看完后你对 event loop 多少有了自己的理解，或者至少有个机会喘口气躺一下。

说实在的，我不确定真的有人能读到这里，Hello？Hello？
