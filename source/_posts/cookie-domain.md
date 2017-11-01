---
title: Cookie：作用域匹配
date: 2017-02-07 23:50:47
tags: cookie javascript
---

> Cookie 相信大家都很熟悉了，至少最常见的场景：用户登录 通常是通过它完成的。这篇博客起源主要由于有天开发，公司分线上和 dev 环境，两个环境调的接口不一样（其中线上接口的 domain 都不同），而接口是要去 `set-cookie` 的，这里出了些容易掉坑的地方。

Cookie 既然可以为我们保存客户端数据，自然它是不能泄漏给其他网站的，否则他人就可以使用你的 cookie 进入系统做一些变态的事情了。浏览器如何保证 cookie 的安全性呢？这里有一个类似作用域的概念，我们设置 cookie 通常是通过如下语法：

```javascript
document.cookie = "name=jiananshi;Domain=shijianan.com;Path=/"
```

这样我们就设置了 Domain 为 `shijianan.com`，Path 为 `/` 匹配这个链接下的 cookie 中的内容：`name=jiananshi`，浏览器在发送请求的时候默认都会带上匹配 Domain + Path 的 cookie 信息。

## Cookie 的匹配规则
Cookie 的匹配分为两部分：域名匹配和路径匹配（下面的规则来自 [rfc6265](https://tools.ietf.org/html/rfc6265)）

### 域名匹配
下面的条件至少匹配一条就可以满足域名匹配：

- Domain 和 URL 完全一致（在匹配算法中它们都会被转为小写再做比较）
- 下面的几个条件都要符合：
  - URL 是 Domain 的后缀（shijianan.com 和 admin.shijianan.com，反过来不行）
  - Domain 结尾可以缺少 %x2E (".") 字符（比如 shijianan.com. 和 shijianan.com 都可以匹配 shijianan.com）
  - Domain 是域名（不能是 IP 地址）
  
### 路径匹配
下面的条件满足其一即可匹配路径：

- cookie-path 和 request-path 完全一致
- cookie-path 是 request-path 的前缀，并且 cookie-path 的结尾是 %x2F ("/") 字符（比如：/src/ 和 /src/a.js）
- cookie-path 是 request-path 的前缀，request-path 同 cookie-path 第一个不匹配的字符是：%x2F (“/“)（比如：/src 和 /src/a.js）

满足匹配规则，浏览器才会在请求时发送对应的 cookie，服务端的 `set-cookie` 也要满足匹配规则才可以成功设置对应 cookie。