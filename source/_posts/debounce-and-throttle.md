---
title: Debounce and Throttle
date: 2016-10-19 09:38:54
tags:
---

几个月前去 TB 面试的时候被问到过关于 debounce，当时没能答上来。回来翻了下源码，发现其实就是我们在页面滚动和自动搜索（用户停止输入后再发送请求）时常用到的概念。另外 Throttle 和 debounce 意思相近但容易让人混淆，这里索性一起写一下。

## Debounce

debounce 是指给定一个时间，只有过了这个时间才会执行这个函数，这期间调用这个函数会重置计时器。实现上也非常简单：

```javascript
function debounce(func, delay=0) {
  let timer;

  function debounced() {
    return setTimeout(_ => {
      func();
      timer = null;
    }, delay);
  }
 
  return function() {
    if (timer) {
      clearTimeout(timer);
    }

    timer = debounced(); 
  };
}
```

debounce 适合的场景有页面滚动加载这种执行频繁但操作可以集中的场景

## Throttle

Throttle，给定一个窗口时间，在这个窗口期内最多只能被执行一次，窗口期间内无论调用多少次这歌函数都不会执行，也不会重置计数器。实现也是非常简单：

```javascript
function throttle(func, wait) {
  let timer;

  return function() {
    if (timer) return;

    func();
    timer = setTimeout(_ => timer = null, wait);
  };
}
```

很明显，throttle 非常适合用于 API 层，某 IP 或用户对于某个请求限制在比如 1 分钟内 100 次，防止无效调用或者恶意攻击（这个要在服务器层面做了）

## 区别

即使代码上差别很大，没遇到过对应场景的人可能还是很难理解和区别它们。我们用举个例子：有一个老司机要发车了 magenet:jsad897dfhdsjkawd08sakl，他可以选择每 10 分钟发一班车车上没人就不发，他也可以每隔 15 分钟发一班车，中间有人来了就重新再等 15 分钟。

所以假设我们在自动搜索里用 throttle，那么用户输入停止后搜出来的结果只会是第一个单词为关键词，这就不符合用户的心理预期了。

## 参考

[underscore.js](https://github.com/jashkenas/underscore/blob/master/underscore.js)