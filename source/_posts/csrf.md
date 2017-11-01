---
title: 如何防范 CSRF 攻击
date: 2017-07-08 16:13:37
tags:
  - csrf
  - security
---

CSRF 的全称是 Cross-site request forgery，中文翻译是跨站请求伪造，简单来说就是用户在登录 A 站点后假冒用户向站点 B 发送请求，如退出用户当前帐号等操作。

下面这张图可以解释 CSRF 攻击的原理：

![csrf](http://oiw32lugp.qnssl.com/2017-07-08-csrf.png)

用户访问站点 A 时，站点 A 用一个图片 html 标签就可以发起 CSRF 攻击：

```html
<img src="//yemengying.com/logout" />
```

假设我们的服务设计了一个接口：`GET /logout` 用户将用户退出登录，那么当他访问 A 站点时，一个 img 标签就足以将他的登录记录干掉（这也是为什么不要把 logout 设计成 GET 请求的原因之一）。

由于 HTTP 协议是无状态的，大部分网站都将用户的信息存储在了 Cookie 中，而图片请求**默认**是会发送用户 Cookie 的，而且图片请求是不会触发跨域的，这使得发起 CSRF 攻击的成本极低。

上面举的例子能造成的影响充其量是对网站多几个 GET 请求，当站点 A 诱导用户进行表单提交或者发送 POST/PUT/DELETE 请求到站点 B 的时候，造成的影响就难以想象了，即使网站阻止了跨域请求也无法防御表单提交。

防御 CSRF 的方法有以下几种：

1. GET 操作永远是幂等的（没有副作用）
2. 拒绝 `Content-Type: x-www-form-urlencoded` 的请求
3. 为每个非幂等的请求增加 token

这里着重讲一下第三点，在用户提交表单前，我们为每个 POST/PUT/DELETE 等非幂等的请求增加一个随机 token 用于验证「用户的表单是在我的站点提交的」，服务端拿到请求后首先验证 token，其次才是处理业务逻辑。

在第一家公司的时候技术栈是 PHP，用户访问页面的时候会在表单中放一个 token 用于验证表单提交：

```php
<?php $csrf_token = md5(123456) ?>
<form>
  <input type="hidden" name="token" value='"' + <?php $csrf_token ?> + '"'>
</form>
```

在饿了么是我们前后端分离的，每次发起 API 请求之前都会请求一个生成 token 的接口用于验证请求源：

```js
// 获取 csrf_token 的接口最好设计成 POST，原因同非幂等其他接口
$http.post('/api/csrf_token').then(token => {
  $http.post('/api/order', { token, params });
});
```

