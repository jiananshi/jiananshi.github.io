---
title: 跨域请求
date: 2015-07-27 14:53:13
tags:
---

这是一个有成熟方案并且内部已经被踩平了的坑了，然而各个项目组依然经常掉坑。<del>其实了解原理以后，主要工作就是前后端一些配置项</del>（还是有很多细节要注意的）。

## 同源策略(same-origin policy)

同源策略是浏览器规范里搞出来的东西，限制了不同源页面之间的资源获取。因为客户端请求的 Header 中会带 cookie，假设 cookie 中包含了登录状态，那么当你像 http://somethingeval.com 发请求的时候，请求服务器就可以获取你的 cookie，冒充你登录你的账号。跨域的情况有以下三种：

+ 不同协议(scheme)
+ 不同端口
+ 域名不同

注意：IE 上不同端口之间的请求不会被视为跨域请求。

假设现在我们在 `http://klam.cc/blog/16` 这个页面发出请求：

| URL | CORS | 
| ------ | --------- |
| http://klam.cc/blog/17 | false |
| https://klam.cc/blog/17 | true |
| http://klam.cc:8080 | true |
| http://something.klam.cc | true |
| http://ele.me | true |

标记为 true 的代表这个请求是跨域的，假设在浏览器上发送不同源的请求，浏览器将会报跨域错误

## JSONP

## CORS

当发送请求时，请求将额外增加一个 Header: Origin，如果服务器的 Access-Control-Allow-Origin 中有配置这个 Origin，那么这个请求就可以发送。

开发时候经常可以我看到过很多开发者将 Access-Control-Allow-Origin 配置为 "*"，或者再笨一点根据上线/测试的状况不断改配置，这都是很愚蠢/低效的做法。配置为 "*" 的潜意识是将 “跨域” 视为麻烦，通过绕过跨域请求来达到目的。另外，虽然 Access-Control-Allow-Origin 头支持配置多个 Origin，但实际上只有一个会奏效。所以正确的姿势是维护一个白名单，比如在 eleme 我们会维护 http://ele.me，https://ele.me 还包括一些测试环境的公网 url。请求过来时取出 Origin 头在数组做查找，匹配到了就将对应 Origin 设置到 Access-Control-Allow-Origin 头信息中，返回给客户端。白名单的设置要遵循权限最小原则，不要搞 *.ele.me 这类东西。"*" 这种 wildcard 式的配置在传输 cookie 时也将失效，客户端通过在请求上设置 `with-credentials: true` 将会在 request 里带上 cookie，服务端配置 `Access-Control-Allow-Credentials: true` 就可以从请求中获取 cookie 了。

#### Preflight request

简单请求(simple request)有两个要求：首先，方法是 GET, POST 和 HEAD 之一。其次，如果 POST 请求发送了数据，数据的 Content-type 是 application/x-www-form-urlencoded, multipart/form-data, 或 text/plain 其中之一。最后，没有自定义 Header 字段。

当浏览器发送非简单请求时，将先发送一个 OPTIONS 请求，根据 OPTIONS 请求的返回判断服务端是否允许当前域访问。在 OPTIONS 请求中，自定义 Header 都将被放到 Access-Control-Request-Headers，比如：Access-Control-Request-Headers: X-Request-With，如果服务端配置了：Access-Control-Allow-Header: X-Request-WIth，那么请求就可以被正常处理。对于 OPTIONS 请求有两点需要注意，OPTIONS 不应当被转发到 Controller 层，判断 Origin 和 Request-Headers 通过后就应当返回 200。另一点是每次跨域请求都发送 OPTIONS，意味着每个请求实际需要一次额外的请求，服务端通过设置 Access-Control-Max-Age 设置 preflight 请求的缓存时间。

## CSRF

## PROXY

配置代理服务器是最稳定的方式，通过服务器反向代理，将原本跨域的请求在浏览器端转化为不跨域的请求，请求到达服务器后由服务器进行转发。

比如: 

`xhr.send('post', 'http://klam.cc/eleme')`

服务器(nginx)上配置:

```shell
location *~ ^klam.cc/eleme{
  rewrite klam.cc/(.*) $1;
  proxy_set_header X-Restapi-From klam.cc;
  proxy_set_header Host ele.me;
  proxy_set_header X-Forwarded-For $remote_addr;
  proxy_pass ele.me;
}
```

请求在客户端的 url 是 `http://klam.cc/eleme` 而经过服务器代理后实际是由 `server_name ele.me` 来进行处理的
