---
title: 使用 Nginx 搭建一个透明的 HTTPS 代理
date: 2017-06-03 15:33:08
tags:
  - nginx
  - https
---

> 前几天看到一个观点「不用 HTTPS 的网站都是耍流氓」，话说的虽然激进了点，但现在不支持 HTTPS 网站的稀有程度堪比 🐼。很多博主通过 hexo 把博客 host 在 Github Pages 服务上，github.io 本身是有证书的，所以访问 yourname.github.io 是有 HTTPS 认证的，这样如果用自己的域名就没法进行 HTTPS 认证了，这篇文章主要说下我是怎么解决这个问题的。

之前写过一篇如何为 Github Pages 配置自定义域名的[文章](https://shijianan.com/2016/12/31/github-pages-https/)，文中推荐了一个国外的服务：CloudFlare，它的原理是把域名的 NS 记录配置到它提供的 DNS 服务，接着做一层路由到源站：

![overview](http://oiw32lugp.qnssl.com/2017-06-03-overview.png)

这个服务唯一的缺点就是在国内的稳定性，前段时间在非北上等地区，比如广州、深圳 CloudFlare 的 IP 是被墙的，这直接导致 host 在上面的站点会无法访问，虽然它提供了诸多好处，但也只能忍痛割爱了。

通常配置 HTTPS 站点需要自己拥有一台服务器，在 web 服务器上配置证书，不过大部分博主用的都是 hexo，自己去搞服务器还要折腾发布和部署，这里我们介绍一种不侵入具体业务，在应用层之前把这个问题解决的方案。

首先服务器还是必须要有的，因为证书的生成和维护必须在服务端完成，但是这并不意味着博客的源文件也要部署到服务器，验证证书的服务器通过后再将请求反向代理到 Github Pages，原理如下图：

![proxy](http://oiw32lugp.qnssl.com/2017-06-03-ssl.jpg)

Pages 这类服务一定是将 {ymy}.github.io 这类域名统一解析到同一个 IP，然后根据 Hostname 路由到不同用户的 github.io 项目上，因此我们的代理服务也要将请求转发到这个 IP 而不是 {ymy}.github.io，因为使用自定义域名 + github pages 很多博主会将 github.io 的域名 CNAME 到自己的域名，直接将请求转发过去会造成循环重定向。

```nginx
location ~ / {
    proxy_pass https://151.101.128.133;
    proxy_set_header Host shijianan.com;
    break;
}
```

`151.101.128.133` 是上面提到的 Github Pages 服务的 IP，而它根据 Host 路由，所以我们要把 Host 头配置成用户的自定义域名，需要注意的是，如果你的 github.io 项目中没有设置 CNAME，pages 也是无法路由你的域名的，最终会拿到一个 pages 服务返回的 404 状态。

最终我们将域名的 A 记录指到这台代理服务器就成功了，这个站点目前正是这么部署的，PING 一下这个站点的 IP，对比你的 github.io 站点看看。

如果提供一个自动为 Github Pages 站点配置 ssl 的服务会有人用么？有前景的话考虑用 go 写一个，不过据说有人学了一年 go 还没写出 hello world，所以也别抱太大期望。