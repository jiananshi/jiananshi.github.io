---
title: HTTP 缓存 
date: 2019-03-08 16:31:31
tags:
  - http
---

ETag、cache-control、expires… 涉及到缓存的 HTTP Header 有这么多用的时候很容易混淆，本篇文章就写一下不同的缓存头是如何工作的。

网上很多文章将缓存分为「强缓存」和「协商缓存」，原始出处不知道是哪里，在 [w3c 草案](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html) 中有这样一段描述：

> …在 HTTP 1.1 中，缓存的目标是在大部分情况下尽量不发送请求，在其他情况下尽量不发送全部的响应，前者减少了网络请求，我们用「过期」机制来实现这个目的；后者节省了带宽，我们用「验证」机制来实现这个目的。

理解了这段话后面的内容就很好懂了，下面我们继续一一浏览 HTTP 缓存头

## 1. Expires 和 Pragma
首先把 expires 和 pragma 这两个头单独拎出来的原因是他们属于 http1.0 的产物，虽然由于自身局限性目前应用较少但是考虑到 1.1 的协议要向后兼容了解一下还是有必要的。

Expires 的值是一个具体的日期，意思是在这个日期前资源都可以被认为是有效的，因为需要要求服务端和客户端时间一致才能高效的工作所以现在基本都在用 cache-control 中的 max-age 来按相对时间做了（cache-control 这个头后面马上会讲）；Pragma 的值只有 `no-cache`，意思并不是字面的不要用缓存的意思，而是在使用缓存前必须同服务器验证有效性，其实跟 `cache-control: no-cache` 一样的所以现在基本也不用了。

## 2. Cache-Control
Cache-Control 是一个功能很多的头，这里我们先只挑最常用的 `max-age`、`no-cache` 和 `no-store` 介绍。相对于 expires 使用绝对日期，max-age 使用的是相对日期（单位为秒），这样就避免了服务器同客户端时间不同步的问题，比如我们为 max-age 设定为 `cache-control: max-age=86400`，客户端在一天之内请求这个文件都无需从服务器获得。

凡事有例外，由于 max-age 的精度太大了，如果在 1 秒内我们的文件出现了变更而 max-age 又还没到过期时间，如何使得客户端及时得到更新呢？这里就涉及到了文章最开始提到的「验证」这个机制，`cache-control: no-cache` 要求缓存在返回资源前必须同服务器验证资源是否过期，因此无论是 expires 还是 max-age 即使在有效期内也必须向服务器发送请求验证资源有效性。而 no-store 的功能是彻底禁用缓存，也就是跳过「过期」和「验证」的机制，每次都像服务器发起请求获得完整的资源。

## 3. Last-Modified 和 If-Modified-Since
上一节提到了在 `cache-control: no-cache` 的时候缓存必须同服务器验证资源有效性，那这个资源是否有效是如何验证的呢？方法之一就是通过 `last-modified` 指定一个具体的日期（这一点同 expires 一样），客户端首次请求后会记录下来这个日期，下一次发起同样的请求的时候会自动加上一个 `if-modified-since` 头（值自然就是这个日期）给服务端，因为日期是服务端生成的，所以这里也不涉及到日期同步的问题，如果值相同服务端只需返回 304 的状态码允许从缓存返回指定资源即可，无需在 body 中返回具体资源。

## 4. ETag 和 If-None-Match
Last-modified 同 max-age 一样，只能精确到秒，前文提到的在 1 秒内更新文件的问题还是无法得到完美的解决，这里就是使用 etag 的时候了。ETag 由服务端生成，通常是根据文件内容或文件名等唯一字段做一个 hash 然后返回给客户端，客户端在「验证」阶段会自动加一个 `if-none-match` 头（值自然就是 etag），同 last-modified 一样服务端在验证 etag 值匹配后返回 304 即可，相较于 `last-modified`，etag 可以实现精度更高的对比。

最后专门建了个仓库来实验文中提到的内容：https://github.com/jiananshi/sample-http-cache

## Reference
[HTTP/1.1: Caching in HTTP](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html)
[HTTP Cache Headers - A Complete Guide](https://www.keycdn.com/blog/http-cache-headers)


