---
title: Javascript Cache API
date: 2017-02-05 10:32:14
tags:
---

> localStorage 和 sessionStorage 相信大家都用过吧，之前学习 ServiceWorker 的时候还接触到了 CacheStorage，这里把自己的理解分享出来

## 创建一个 CacheStorage 实例
首先我们要检测浏览器是否支持 Cache API：

`assert(‘caches’ in window)`

接下来我们要创建一个 CacheStorage 对象供我们读写缓存：

```javascript
caches.open('project-1.0').then(cache => {
});
```

`open` 函数第一个参数接收一个字符串（key），推荐的命名方式是在其中加入 version 字段以供 breaking-change 的变更带来的前端缓存更新问题，注意 `caches` 的 API 返回的都是 Promise。

## 添加和删除缓存的响应
Cache API 最大的用处在于可以指定请求并且支持配置自定义的响应：

```javascript
caches.open('project-1.0').then(cache => {
  cache.add('https://shijianan.com');
});
```

这段代码运行后会立刻发送一个 get 请求到指定的 url 并将结果缓存在 cacheStorage，`add` 方法除了接收字符串外也接受 `Request` 对象，这样如果请求还有一些额外的头和 queryString 等等也都可以匹配。如果想要为请求指定响应可以这么做：

```javascript
caches.open('project-1.0').then(cache => {
  cache.put('https://shijianan.com', new Response('fake response'));
});
```

## 其他 API
`Cache.addAll`：可以为一组请求添加响应
`Cache.match`：根据请求获取 CacheStorage 中的响应
`Cache.delete`：删除缓存
`Cache.keys`：列出所有的 Cache key（注意这个 API 返回的也是 Promise）

## 注意
- Cache.put, Cache.add, 和 Cache.addAll 只支持 GET 请求
- Cache API 只能在 HTTPS 站点使用
- Cache API 会无视 HTTP 缓存头