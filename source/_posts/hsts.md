---
title: HSTS 是什么
date: 2017-06-16 07:32:42
tags:
  - https
  - hsts
---

网站开启 HTTPS 之后，通常我们希望将 80 端口通过 HTTP 协议访问的流量也导到 HTTPS 443 端口，自此在服务或站点上就不用为劫持和中间人费心了。

要做到这点很简单，在服务器上将 80 端口的请求 302 到 443 端口即可：

```nginx
server {
    listen 80;
    server_name shijianan.com www.shijianan.com;
    return 302 https://$host$request_uri
}

server {
    listen 443 ssl;
    ...
}
```

然而这个方案有两个弊端：一是性能上的略微损耗，第一次为 HTTP 请求建立的 TCP 三次握手是没有意义的，二是通过 HTTP 访问的用户依然有可能被劫持，这可能会造成某些用户永久性无法访问你的站点。

HSTS 就是为了解决这个问题诞生的[RFC 6797 - HTTP Strict Transport Security (HSTS)](https://tools.ietf.org/html/rfc6797)，当用户通过 HTTP 访问支持 HSTS 的站点时，浏览器内部就会将这个请求重定向到 HTTPS

![redirect](http://oiw32lugp.qnssl.com/2017-06-15-232131.jpg)

为了让站点支持 HSTS，我们首先需要为所有响应添加一个响应头，格式如下：`Strict-Transport-Security: max-age=expireTime [; includeSubDomains] [; preload]`

HSTS 生效后发往站点的 HTTP 请求就会被 307 到 HTTPS，但是前提是 HSTS 生效，因此如果用户首次是通过 HTTP 访问这个站点，它依然不会被浏览器内部重定向，这时就需要用到 **HSTS Preload List**

HSTS Preload List 是浏览器内置的一份名单，这个名单中的域名即使通过 HTTP 访问，依然会被浏览器内部重定向到 HTTPS，目前这个名单 Chrome、Firefox、Safari、IE 11 和 Microsoft Edge 都在使用，个人站长也可以前往 https://hstspreload.org/ 检查自己的站点是否符合要求并提交申请添加进列表。需要注意的是，一旦站点上了 HSTS，就不能撤销了，用户总是会通过 HTTPS 来访问你的站点，这一点我觉得技术是不可能倒退的，因此这个站已经提交申请并处于 pending 列表中了。

最后附一份 nginx 的 HSTS 配置：

```nginx
server {
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location / {
      // 如果 location block 中有 add_header 是不会继承 server 中的 add_header
      add_header Access-Control-Allow-Origin *;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    }
}
```

