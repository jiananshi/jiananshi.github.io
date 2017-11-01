---
title: Javascript 自定义事件
date: 2017-01-12 21:51:24
tags:
---

有天在做一个联动的组件的时候遇到一个场景，iframe 中的某些变化需要同步更新外层 html 中的某些元素，怎样可以通知到呢？

我们可以在外层的页面中通过 contentDocument 拿到 iframe 里的 window 对象，然后由于变化不一定是原生内置事件，比如鼠标键盘等交互，这里我们可以通过 `CustomEvent` 来创造一个自定义事件冒到 window 上：

```javascript
var myEvent = new CustomEvent('myEvent');
```

之后通过在 dom 元素上调用 dispatchEvent 来触发它：

```javascript
var $btn = document.querySelector('#myBtn');
$btn.addEventListener('myEvent', handler);
$btn.dispatchEvent(myEvent);
```

如果需要 event 传递一些数据可以在 event 创建的时候传入一个 detail：

```javascript
var myEvent = new CustomEvent('myEvent', {
  detail: 'data'
});
// 需要注意的是：
// 自定义事件对象一旦创建
// 就不能再传入数据了
```

那么 CustomEvent 的兼容性如何呢？这里用 caniuse 查一下：

![caniuse](http://oiw32lugp.qnssl.com/2017-01-12-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-12%20%E4%B8%8B%E5%8D%8810.51.27.png)

可以看到 IE 需要 11 以外其他主流浏览器都支持良好

## 相关资源
[CustomEvent - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent)