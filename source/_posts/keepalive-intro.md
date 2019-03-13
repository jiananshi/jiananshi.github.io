---
title: keepalive 简介
date: 2018-02-22 17:59:28
tags:
  - keepalive
  - backend
  - tcp
  - frontend
---

![image](//pic.yupoo.com/jiananshi/880e7b33/0a1ecd3b.jpg)

## 前端关注的 keep-alive
如果你关注过「缓存」这件事，写前端的时候一定会注意到诸如：Cache-control / etag / connection… 诸如此类的 HTTP 头，如下图所示：

![image](//pic.yupoo.com/jiananshi/d32772d5/8187e6ea.jpg)

（注：keep-alive 属于 HTTP/1.1 的内容，h5.ele.me 很多资源上了 h2，这里特意选了一个 HTTP/1.1 的请求做例子）

我们都知道 HTTP 请求是很昂贵的，在没有开启 `connection: keep-alive` 的情况下，对于 `/style.css` 和 `/script.js` 这两个文件，浏览器将不得不为每一个请求创建一个 TCP 连接。同时由于在浏览器中要保证资源的加载顺序，我们不得不等待前一个 TCP 请求响应后再请求下一个资源，除了 TCP 创建/销毁 的开销，阻塞式加载也会成为性能瓶颈。

而在 HTTP/1.1 中 `connection: keep-alive` 默认是开启的，得益于此，对于 `/style.css` 和 `/script.js` 这两个文件我们只需要一个 TCP 连接就够了。不仅如此，[HTTP pipeling](https://en.wikipedia.org/wiki/HTTP_pipelining) 可以让同域名下的请求合并在一起发送无需等待前一个请求的响应，流程类似下图：

![image](//pic.yupoo.com/jiananshi/c153a355/ae45c76b.jpg)

## 后端关注的 keep-alive
抛开 API Layer 层，后端调用其他内部服务没必要使用 HTTP 这种开销比较大的协议，通常通过 rpc 调用就够了，因此接下来我们讨论的是基于 TCP 的 keep-alive 而不是 HTTP（七层）的 keep-alive。

在通信双方建立好 TCP 连接以后就可以开始通信了，网关会检测 TCP 连接是否活跃，在长时间没有数据传输的情况下会主动断开这个 TCP 连接，如下图：

![image](//pic.yupoo.com/jiananshi/47e142a5/004418e6.jpg)

很多服务都有连接池的概念，这种场景下我们需要创建一组 TCP 连接并且在使用的时候才开始数据传输，如果连接闲置一段时间会被主动断开就无法实现这个需求了（当然也可以重新创建一个），解决方案就是客户端通过不间断的发送心跳包告诉服务端这个连接依然有效（相应的，服务端需要回一个数据为空并开启了 ACK 的包），避免网关或另一端通信方断开连接：

![image](//pic.yupoo.com/jiananshi/35eecf20/5d249321.jpg)

之所以会关注到 TCP keepalive 还要从之前遇到过一个线上问题说起：DBA 半夜跑脚本，CPU 占用 100%，机器资源几乎被耗光，客户端连接池中连接心跳包检测失败，在超过 retry_times 后连接池中所有连接，早上流量进来的时候才发现连接池中的连接均不可用，直接导致了业务不可用。

### 参考

[HTTP/1.1 rfc2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html)
[TCP keepalive](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html)
