---
title: Github Pages 配置 https
date: 2016-12-31 06:37:07
tags:
---

前言：昨天去喝酒了，博客断了一天，误事啊。。。

[Github Pages](https://pages.github.com/) 可以为 Github 任何的项目生成一个静态的网站，`https://shijianan.com` 就是部署在 pages 上的，本篇文章并不想涉及如何部署等细节，详情自行去官网查看就好。后来才了解到 pages 还支持自定义域名，于是在 dnspod 上把自己的域名解析到了 `https://shijn.github.io` 上，数分钟后访问自己的域名就可以到 github 的服务器上了，破费。

![https logo](https://oiw32lugp.qnssl.com/2016-12-30-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-12-31%20%E4%B8%8A%E5%8D%887.29.03.png)

但是等等，原来 github.io 域名带的小绿锁，也就是 https 证书失效了。在我们创建 pages 的时候，github 的证书认证了 \*.github.io，可以支持用户通过 https 访问。我们自己的域名解析到 github 必然没有证书认证，不支持 https 也是合情合理的了。又不能上 github 服务器用 letsEncrypt 这种工具生成证书，怎么办呢？

我们可以通过配置一台代理服务器解决，服务器至少要满足两个要求：

1. DNS 解析，将访问我们域名的请求解析到这台服务器的 IP
2. CNAME 配置，将访问本台服务器的 http/https 流量重定向到 github pages
3. 为我们的域名生成一个证书

自己去折腾这一套的话 DNS 部分维护起来太不稳定了，[CloudFlare](https://www.cloudflare.com) 就在做这件事，而且是完全免费的。具体配置方式可以看 CloudFlare 的[官方博客](https://blog.cloudflare.com/secure-and-fast-github-pages-with-cloudflare/)

![cloudflare rule](https://oiw32lugp.qnssl.com/2016-12-30-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-12-31%20%E4%B8%8A%E5%8D%887.34.46.png)

解析到 CloudFlare 后，我们可以配置 HSTS 和 Page Rule，让用户强制访问 https，并且保证站点所有静态资源走 https。除此之外，CloudFlare 还提供流量分析统计、自动压缩、WebP 优化、CDN 缓存 等等等等，有兴趣可以折腾一下（真没收 CloudFlare 的广告费 😂  ）。





