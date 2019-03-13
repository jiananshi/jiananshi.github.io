---
title: Event loop 是什么
date: 2019-03-05 17:15:37
tags:
  - javascript
---

终于把 Youtube 上收藏了好几年的 [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ) 看了，所以记录下来。

在开始了解 Event loop 之前，首先需要对 JavaScript 的执行栈（call stack）有一个基本的了解，如下的代码：

```js
function a() {}

function b() {
  a();
}

function c() {
  b();
}

c();
```

浏览器在运行这段代码的时候会按照 c -> b -> a 的顺序将放入 call stack 中，然后再在对应的函数运行完成后依次推出，整个顺序是按照先入后出的（FILO）。

了解了这一点之后来看下面这张图：

![img](http://pic.yupoo.com/jiananshi/c6561299/d9e99db4.png)

JavaScript 是一门单线程语言，如果我们在发送网络请求、更新 DOM 的时候都是同步的话就会阻塞，也就是说 call stack 卡住了我们无法往里放入新的代码去执行，界面也无法响应用户的交互。而 event loop 使我们避免了这一情况，在执行图中所示的浏览器 API 的时候，代码并非立刻执行而是被放到了一个任务队列中（FIFO），当 call stack 被清空以后浏览器会检查队列中的任务并依次执行他们（注意这里的顺序是先进先出），至于 Promise 和 MutationObserver 那些 microtasks 用的是另一个队列，具体可以看之前翻译的一篇[文章](https://shijianan.com/2017/03/18/tasks-microtasks-queue-schedule/)。
