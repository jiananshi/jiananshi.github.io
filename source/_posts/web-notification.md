---
title: 开启 Web 桌面通知
date: 2017-02-08 22:04:28
tags: notification javascript
---

通知在手机系统上是很常见的功能，比如我们使用微信的时候，退出应用后有消息进来系统会推送通知给用户，这里涉及到两部分：推送 和 通知，本篇文章主要来看一下 Web 通知（顺便给自己立个 flag，下一篇会写相对复杂一点的 Web 推送）。

## 申请权限
第一步依然是检测浏览器是否支持 Notification API：

`Reflect.has(window, ‘Notification’)`

截止到本文成文浏览器的支持情况如下图：

![support](http://oiw32lugp.qnssl.com/2017-02-08-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-08%20%E4%B8%8B%E5%8D%8810.23.11.png)

支持 Notification API 的浏览器会在全局暴露一个 `Notification` 对象，向用户发送通知首先要获取用户的同意（否则无良广告商或者开发者使用不当会给用户体验带来灾难性影响），通过调用 `Notification.requestPermission` 获取用户授权，不同于一般的 API，这个 API 会有类似 BOM 的交互，浏览器中会弹出一个窗口（如下图）请求授权：

![request permission](http://oiw32lugp.qnssl.com/2017-02-08-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-08%20%E4%B8%8B%E5%8D%8810.26.36.png)

这个函数的参数是一个回调函数，参数就是用户授权的结果，有三种：

| 名字 | 详细 |
| ——— | ——— |
| granted | 已获取用户授权 |
| denied | 用户拒绝授权 |
| default | 未知用户授权结果 |

申请授权后用户以后再次进入这个页面可以通过 `Notification.permission` 来获取授权状态决定是否再次授权或是推送消息。

## 创建通知
桌面通知通过 `Notification` 的实例创建：

`let notice = new Notification(‘鸡年大吉’)`

注意 notice 除了用户点击关闭不会自动清除，所以最好在一段时间后手动清除：

`setTimeout(notice.close.bind(notice), 500)`

`Notification` 的构造函数除了接受通知内容外，还接受一个对象用来配置通知：

```javascript
let notice = new Notification('鸡年大吉', {
  icon: 'https://shijianan.com/images/logo.png', // 通知图标
  body: '皮皮虾我们走！', // 通知内容
  lang: 'zh-CN', // 语言
  dir: 'ltr', // 文本方向（实测没卵用...）
  tag: 'newyear' // notification 的标签，可以用来作为标示，以备处理同一类 notification
})
```

## 通知 Hook
在实例被创建出来后，我们可以通过几个钩子函数来做一些事情：

- onshow() 消息显示
- onclick() 点击事件
- onclose() 关闭事件
- onerror() 内部报错

下一篇我们会聊一下使用 Push API 实现消息推送，结合通知和之前提到的 ServiceWorker 基本就能模拟 APP 的用户体验了，感觉 Web 要起飞了。
