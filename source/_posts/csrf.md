---
title: CSRF 攻击是什么
date: 2017-07-08 16:13:37
tags:
  - csrf
  - security
---

CSRF 的全称是 Cross-site request forgery，中文翻译是跨站请求伪造，绝大部分网站会使用 cookie 保存用户当前的登录状态，在我们请求对应站点 API 时 cookie 也会自动携带，攻击者就是利用了这一点从站外向目标站点发出请求，利用 cookie 自动发送的特性发起攻击。

举个例子：

```html
<img src="//shijianan.com/user/logout" />
```

假设站点的登出功能设计为 get 请求，那么访问这个页面的用户此时就会被登出。那么避开 get 请求是否就可以了呢？再看一个例子：

```html
<form method="post" action="//shijianan.com/user/logout">
	<input type="hidden" name="name" value="jiananshi" />
</form>
<script>
  document.querySelector('form').submit();
</script>
```

利用 JavaScript 用户一打开这个页面就会自动向服务端发送一个 post 请求，因此防御的方法不能单纯从 HTTP method 出发，目前主流的做法一是验证 HTTP Referrer（跨域请求验证 origin），二是发起请求前后端先生成一个 csrf_token 给前端，调用 API 时同时验证 csrf_token，通常的做法是提供一个接口获取当前 csrf_token（若过期则生成新的），前后端未分离的业务可以在后端渲染模版的时候在前端埋入。
