---
title: 为你的 Disqus 搭建代理服务
date: 2017-01-02 14:57:55
tags:
---

很多用 hexo 的朋友也同时使用了 Disqus 作为博客的评论系统，前段时间（得有大半年了 😂 ) Disqus 被墙了，导致女友博客的评论瞬间掉低，一起讨论下想出了一套曲线救国的方案：通过代理（正向）服务调用 Disqus 服务

代理可以理解为用户去调用 A ，A 帮助它去调用了 B，最后将 B 的返回结果还给用户。这个场景正是因为国内调不通国外服务，所以通过在境外（或者香港机房）架设一台 VPS 调用 Disqus 接口来解决，由于 Disqus 开放了 [API](https://disqus.com/api/docs/) 使得这个方案可行。

首先我们要部署好我们服务器上的 API 服务，当然这台服务器一定要 Ping 的通 Disqus，官方提供了一个 [SDK](https://github.com/disqus/disqus-python) 所以推荐使用 python 完成接口编写，或者直接 clone 现成的：[disqus-sdk](https://github.com/shijn/disqus-proxy) 执行 `pip install` 然后用 supervisor 起了就欧了，至此，服务端部分部署完毕。

前端相对任务较重，我们需要在合适的时机调用代理服务器，并且将返回渲染到页面上，当时写了个 [DisqusJS](https://github.com/shijn/DisqusJS) 可以拿来用，考虑过评论内的图片、响应式以及加载策略等等因素，效果可以去 [giraffeHome](http://yemengying.com) 看，目前还有很多可以完善的地方，比如登录、压缩、分页等等一直拖着没做，欢迎督促和 PR。