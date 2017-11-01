---
title: Nginx HTTPS 性能调优
date: 2017-06-18 07:32:35
tags:
  - nginx
  - https
  - ocsp
---

相比于普通的 HTTP 请求，HTTPS 请求需要占用更多的资源用于 TLS 协议、证书的验证、请求实体的加密验证等等，最直观的体验就是访问 HTTPS 的网站会比访问 HTTP 协议的网站要慢（不支持 http2 的话）。

本篇文章我们来看一下如何在服务端优化 HTTPS 请求，文中以 Nginx 为例。

**Session Cache** 和 **Session Ticket**

Session Cache/Ticket 会在服务端缓存 TLS 握手信息的 Session 供其他进程和后续的连接使用，区别在于 Session Cache 将 Session 保存在服务端，而 Session Ticket 是存储在客户端的。

在 nginx 中对应的配置如下：

```nginx
http {
    # [shared:name:size]
    # shared 表示多进程间共享 Session Cache
    # size 默认单位是 byte
    ssl_session_cache Shared:SSL:10m;

    ssl_session_tickets on;

    # 配置 session 的有效期
    ssl_session_timeout 10m;
}
```

**OCSP stapling**

之前的文章里提到过，大部分站点的 HTTPS 证书都是 intermediates Certificates 颁发的，客户端（浏览器）在请求服务端的时候会去顺着证书链验证服务端的证书，这部分处理会拖慢 TLS 握手，OCSP stapling 这一策略就是为了优化这个问题。

OCSP（Online Certificate Status Protocol）是在线查看 CA 证书的状态，OCSP stapling 要求服务端在建立 TLS 握手的同时返回服务端缓存的 OCSP 记录避免客户端发起请求验证的损耗，响应的服务端也会缓存 OCSP。

启用 OCSP stapling 有以下几个优势：
- 只在客户端需要的时候发送 OCSP
- 验证 CA 不会带来额外的 HTTP 请求
- 在服务端验证减小了收到恶意攻击的可能性

对应的 nginx 配置如下：

```nginx
http {
    ssl_stapling on;

    # 是否验证响应的 OCSP，开启这个配置后需要配置 ssl_trusted_certificate
    # 由于我们用的是 Let's Encrypt 证书，响应的 OCSP 中并没有证书，所以无需配置
    ssl_stapling_verify on;
    ssl_trusted_certificate /home/file/ca.cert
}
```

重启 nginx 之后即可通过 ssllabs 查询服务端 OCSP stapling 状态

![OCSP stapling](http://oiw32lugp.qnssl.com/2017-06-17-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-18%20%E4%B8%8A%E5%8D%887.25.25.png)

======== 分割线 ========

昨天被鸽了牛蛙，今天被割了火锅，今天已经啥都不想干了。。。


