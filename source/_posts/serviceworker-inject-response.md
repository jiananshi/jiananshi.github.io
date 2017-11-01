---
title: Service Worker：拦截请求
date: 2017-01-29 13:42:01
tags: javascript ServiceWorker
---

Service Worker 为浏览器提供了缓存、拦截请求、推送消息、离线应用等帮助开发者为 webapp 提供接近 native app 体验的功能，本文主要讨论拦截 HTTP 请求部分

## 创建一个 serviceWorker
`serviceWorker` 的代码需要放置在一个 JS 文件里，通过 `serviceWorker.register` 注册后才可以使用，在注册之前我们首先需要判断浏览器是否支持：

`if (‘serviceWorker’ in navigator) { /* do things */ } `

register 方法接收两个参数：serviceWorker 脚本的 url 和额外的可选参数：

`serviceWorker.register(scriptURL, options?)`

options 目前仅支持一个参数：`scope`，指定这个 serviceWorker 可以控制的 path，通常是一个相对路径，比如：`{ scope: ‘/‘ }`。

## ServiceWorker 的生命周期
从我们写下 `serviceWorker.register(‘path/to/file.js’)` 开始，这个 serviceWorker 的运行经历了如下过程：

### 1.1 Download
### 1.2 Install
这个阶段主要用于主要用于为 worker 做一些初始化，例如离线缓存，`event.waitUntil` 接收一个 Promise，在它 resolve 后才会进入下一阶段。
### 1.3 Activate
这一阶段你可以调用 `self.clients.claim` 将当前这个 serviceWorker 设置为所有在它作用域里的客户端的 active worker（虽然 serviceWorker 工作于另一个线程，但是在一个页面只能同时归属一个 serviceWorker 控制，尝试去创建一个新的 serviceWorker 会发现它们永远处于 `pending` 状态）。

![life cycle](http://oiw32lugp.qnssl.com/2017-01-29-042809.jpg)

## 事件处理
ServiceWorker 脚本中会有一个全局对象：`self`，它上面会有很多内置的属性和方法，其中 `addEventListener` 可以提供我们对生命周期和事件的控制，不同于 dom 的事件，它的事件有：install、activate、fetch、notificationclick 等等，本节我们看一下如何使用 fetch 事件对请求伪造响应。

在 serviceWorker activate 后，所有作用域下的 fetch 请求都会触发 `fetch` 事件，回调函数的 event 参数有一个 `respondWith` 方法，它可以接收一个 Response 对象并返回给客户端，这样我们就可以轻松实现自定义 HTTP 响应了，原理如下图：

![hijack response](http://oiw32lugp.qnssl.com/2017-01-29-053048.jpg)

ServiceWorker 代码部分：

```javascript
self.addEventListener(‘fetch’, e => {
  e.respondWith(
    new Response(‘serviceWorker intercept’, {
      headers: { ‘Content-type’: ‘text/plain’ }
    }));
});
```

## 其他
- ServiceWorker 必须在对应的 fetch 请求发送之前注册，否则不会请求。
- 当我们的作用域下（比如：’/a’）有多个 serviceWorker 满足时，最符合的会应用（’/a’ 就比 ‘/‘ 优先级高）。
- 在 ServiceWorker 中你不能访问 DOM、localStorage、XMLHttpRequest 等等，但是你可以使用 fetch 和 indexdDB。