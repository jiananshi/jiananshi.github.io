---
title: HTTP 缓存失效的计算
date: 2017-05-25 07:53:41
tags: 
  - http
  - cache
---

HTTP 1.1 中要求服务端为每个请求设置一个 `Date`  响应头，表示这个资源生成的时间。当从缓存中获取资源时，HTTP 1.1 会在每个资源上设置 `Age` 头，它的值是一个数字，表示资源已经存在的时间（或是距离上一次同服务器验证有效性的时间）

下图是我在百度贴吧的请求里截了张图：

![screencast](https://oiw32lugp.qnssl.com/2017-05-25-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-25%20%E4%B8%8A%E5%8D%888.45.39.png)

根据前文提到的信息，这个资源的创建时间是 2017 年 5月 25 日 00:24:14，资源存活时间是 1190059（不到二十分钟的样子）。需要注意的是服务器的时间不一定同客户端的时间一致，服务端和缓存客户端最好使用 [NTP](http://www.ietf.org/rfc/rfc1305.txt) 或其他类似的协议保持整体时间的一致性。

现在我们有两种方式计算资源已存活的时间，下面的公式里我们用 `date_value` 表示 Date 头，而用 `age_value` 表示 Age 头，客户端时间用 `now` 指代：

1. now - date_value
2. age_value

这两个公式可以基于一种保守的策略合并为一个公式：

```
corrected_received_age  = max(now - date_value, age_value)
```

由于存在网络延时性的问题，资源的实际存活时间通常会比计算出来的要小，而根据不同客户端的网络环境差值会相差很大，因此我们把请求发起的时间 `request_time` 也计算在内：

```
corrected_initial_age = corrected_received_age
                      + (now - request_time)
```

有了资源的存活时间，接下来我们只要比较资源的有效期 `freshness_lifetime` 和资源的存活时间就可以得出资源是否过期的结论了，下面我们用 `expires_value` 表示 Expires 头，而用 `max_age_value`  表示 Cache-Control 的 `max-age` 子指令。

因为 max-age 比 Expires 头的优先级要高，如果存在 max-age 的话公式如下：

```
freshness_lifetime = max_age_value
```

否则则根据 Expires：

```
freshness_lifetime = expires_value - date_value // 上文提到的 Date 头
```

如果这两个头都不存在的话（还包括 Cache-Control: s-maxage，总共三个） 也有一种算法可以计算缓存失效时间，下面我们用 `last_modified_value` 表示  Last-Modified 头：

```
freshness_lifetime = (now - last_modified_value) * fraction
```

fraction 是一个不大于一的数，通常会设为 10%，也就是说一个资源距离上一次更新有 10 天的话，那么它的缓存时间就是 1 天（10 * 10%），只要资源存活时间在一天之内那么它就是没有过期的。

## 参考资料
[rfc2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html)
