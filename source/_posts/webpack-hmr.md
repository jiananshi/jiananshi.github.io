---
title: Webpack Hot Module REPLACEMENT(HMR)
date: 2017-01-13 11:35:24
tags:
---

记得刚开始写前端的时候有个很受欢迎的插件叫 LiveReload，客户端还要付费很多人也用，后来前端编译（Grunt、Gulp）流行起来以后基本都用内置插件了。最近写题老师的项目头一次接触到了 HMR，之前只是听过，在使用时发现通过 Websocket 的推送实现更新 + 单页应用重现渲染比 LiveReload 刷新在体验和效率上有了极大的提高。

先来看看 Vue 提供的 HMR 使用效果图：

![vue-hmr](https://camo.githubusercontent.com/55378bb66e697180c6d13d42e3767d99e3a17c6f/687474703a2f2f626c6f672e6576616e796f752e6d652f696d616765732f7675652d686f742e676966)

HMR 实时监测到代码的更新并且触发 Vue 组件重新渲染，通过 Websocket 通信所以无需 Reload，平时开发能节省多少 `cmd + R` 的时间啊。

## 用法
HMR 是一个选择性使用的插件，所以你可以在应用层用代码获取 HMR 的句柄，举个例子：

```javascript
if (module.hot) {
  // HMR 是否触发
}
```

HMR 使用消息冒泡的机制，组件更新后会发射一个事件，这个事件一直向上冒泡知道被捕获（或者到达根节点）。

![hmr theory](http://oiw32lugp.qnssl.com/2017-01-13-063052.jpg)

下面我们来捕获 HMR 的更新事件并重新渲染视图：

```javascript
if (module.hot) {
  module.hot.accept(); // 接收事件并阻止冒泡
  require(‘./component’).render(); // 重新渲染视图
}
```

## 清除副作用
举一个不那么恰当的例子，假设我们现在想要热替换 CSS 模块，我们除了需要插入新的 style 标签，还需要移除老的（否则会不断的有 style 标签被叠加）HMR 提供了一个 dispose 的方法供我们在模块被更新之前清除代码造成的边际影响（side effect）：

```javascript
if (module.hot) {
  module.hot.dispose(function() {
    document.body.removeChild(oldStyleTag);
  })
}
```

## References

- [Understanding Webpack HMR](http://andrewhfarmer.com/understanding-hmr/)
- [Webpack HMR 官方文档](https://webpack.github.io/docs/)