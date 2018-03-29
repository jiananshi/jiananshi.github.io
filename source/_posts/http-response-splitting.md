---
title: HTTP Response Splitting 
date: 2018-03-29 08:14:13
tags:
  - security
  - http
---

前阵子安全部门报过来了一个非常有意思的漏洞，在我们支持 HTTPS 后，HTTP 访问都会被 301 到 HTTPS，在服务端配置如下：

```nginx
server {
  # ...
  server_name example.com;
  return 302 https://$host$request_uri;
}
```

乍一看是非常简单的配置，但其中有一个容易忽视的地方是，我们直接把用户输入 `$request_uri` 添加到了 url 中，这就给客户端带来了注入的机会。

再进一步了解这种攻击方式之前，首先我们需要知道 HTTP 1.1 是通过换行符分割 header 和 body（通过两个换行符表示），格式如下：

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 194
Connection: keep-alive

<html>
  <body>
    <h1>hello world</h1>
  </body>
<html>
```

设想这样一种可能性，假如用户可以把换行符插入到 `$request_uri` 中，那么他就可以轻易的操纵 HTTP response 的返回了，因此如果把换行符编码为 `%0D%0A` 并放在 url 中，寻找到这种重定向的逻辑就可以实施攻击了，比如上文我们提到的 301 到 https：

```
curl http://shijianan.com%0D%0ASet-Cookie:X-CLRF-ATTACK=true
```

HTTP response 将会变成：

```
HTTP/1.1 301 Moved Permanently
Content-Type: text/html
Connection: keep-alive
Location: https://shijianan.com
Set-Cookie: X-CLRF-ATTACK=true
```

这里由于这个接口是个重定向逻辑，相应的能做的事情也比较少，如果是 200 之类的接口就可以进行跨站攻击、钓鱼等等攻击了。

（附注：目前主流的 Web 服务器都修复了这个 bug，所以对本站发出这样的请求只会是字符串编码后的 `%0D%0A`）
