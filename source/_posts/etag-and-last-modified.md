---
title: Etag 和 Last-Modified 的优先级
date: 2019-03-10 22:24:29
tags:
  - http
  - cache
---

[上一篇文章](https://shijianan.com/2019/03/08/http-cache/)中了解了常用的 HTTP 缓存头部，不难发现在不同工作阶段有的头部的工作会相互冲突，当发生冲突时服务器应当如何处理呢？

## 1. 过期和验证
过期的优先级高于验证，如果 expires 和 cache-control 存在且没有过期，那么客户端并不会对 etag 和 last-modified 向服务端发起验证（expires 属于 HTTP 1.0，这里不针对他的优先级做赘述）

## 2. Etag 和 Last-Modified
[rfc2616](https://www.ietf.org/rfc/rfc2616.txt) 中专门有一章 13.3.4 讲解他俩之间的关系，总结来说如下：

- 服务端应该发送 etag 
- 服务端应该发送 Last-Modified
- 客户端应当验证 etag
- 当只有 Last-Modified 的值的时候客户端应当验证
- 如果 etag 和 Last-Modified 都有，两个都要验证
