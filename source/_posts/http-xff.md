---
title: HTTP 头 X-FORWARDED-FOR
date: 2017-07-01 17:08:11
tags:
  - http
---

Disqus 被墙以后配置了一个反向代理去调 Disqus api，那个服务到现在运行的还不错，美中不足的是经过了一层反向代理后我们丢掉了用户的 IP 信息，最终提交给 Disqus 的只有转发服务器的 IP，这样所有用户的请求都来自于同一个 IP，假如要展示访客地区分布就捉🐔了。

反向代理对于用户来说是透明的，客户端并不知道这个接口后面有几层代理，而最后一层代理甚至有可能不知道真实用户的 IP，他只能拿到上一层代理的 IP，要解决这个问题就涉及到一个 HTTP header：X-FORWARDED-FOR。

记得最开始看到这个头还是在一哥的 nginx 配置中，「X」开头的 HTTP 头通常是自定义的存储新的地方，所以当时并没有太当回事，现在才发现这个头居然都被写到[规范](https://tools.ietf.org/html/rfc7239)里了。

它的作用是记录每一次转发经过的 IP，假设客户端的请求如下：

Client -> ProxyA -> ProxyB -> API Server

那么 API Server 拿到的 X-FORWARDED-FOR 应该是：

Client.IP, ProxyA.IP

ProxyB 的 IP 就不用再写了，因为他就是发起请求的 IP，建立完 TCP 链接就知道了。

在 Nginx 配置中，一个常见的误导是将这个头配置为 $remote_addr，这么做会导致前面转发的 IP 丢失，nginx 有一个 $proxy_add_x_forwarded_for 可以供我们使用，他会自动在 X-FORWARDED-FOR 头后追加 $remote_addr，如果 XFF 头不存在的话则设置成 $remote_addr。

```
server {
  location ~ {
    proxy_pass https://shijianan.com;
    proxy_set_header $proxy_add_x_forwarded_for;
  }
}
```