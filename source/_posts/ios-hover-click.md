---
title: iOS 诡异的两次点击跳转
date: 2016-12-29 07:54:27
tags:
---

前几天在为 [Duang](https://github.com/eleme/duang) 编写文档的时候，发不到 github pages 后发现在手机上，页面上的链接要点击两下才能跳转，桌面端 Safari 就没问题，试了下题老师的 Android 都可以，诡异。Google 了一波并结合样式看了一下才豁然开朗，在 iOS 设备上，如果某些内容是基于 `:hover` 显示的，那么在 iOS 移动端设备上，它会对第一次 tap 展现出 :hover 的效果，第二次才会触发 click。下面的样式就会出现这种情况：

```css
a:hover::after {
  content: 'hover';
}
```

解决方法也非常简单，在 iOS 设备上将 `touchstart` 事件转为 click 即可

```javascript
element.addEventListener('touchstart', () => element.click());
```

## Reference

[The Annoying Mobile Double-Tap Link Issue](https://css-tricks.com/annoying-mobile-double-tap-link-issue/)