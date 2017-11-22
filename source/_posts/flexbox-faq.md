---
title: Flex 的一些常见问题
date: 2017-11-23 02:40:56
tags:
  - css
  - flexbox
---

记得 Flexbox 刚刚开始有人用时，除了兼容性问题大家一致觉得这个特性节省了前端写页面的时间，记得移动端的美食墙（目前已经这个功能已经下线了）就是利用了 Flex 可以在垂直方向上排列元素的特性来写的。使用 Flexbox 免去了往常写 float 要处理高度坍塌，有的场景要考虑到 BFC（Flex container 本身会创建一个 FFC - Flex Formatting Context)，脱离文档流后的相对高度等等问题。随着 Flex 的深入在日常开发中也遇到了一些匪夷所思的问题，本文记录一下。

## Flex-item 的高度被强制拉伸对齐
假设我们的 flex-container 中有两个 flex-item，当其中一个 flex-item 设置高度后，另一个 flex-item 会被强制拉伸至同样的高度，下面是一个例子：

<script async src="//jsfiddle.net/jiananshi/rgr0wsfk/1/embed/html,css,result/"></script>

之所以会这样是因为 flex 的一个属性在作怪：`align-items`，在实际开发中我发现很多时候我们习惯写下这样的 mixin：

```css
.flex {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
```

由于 align-items 总是被设置成 center 导致他的默认值其实是 `stretch` 很难暴露出来，这个问题除了通过在 flex-container 上改变 align-items 的值之外，也可以通过在不想要 `stretch` 的 flex-item 上设置 `align-self` 来调整。

## Flex-item 的大小同声明不一致
首先要说明的是 flex 默认是不换行的，除非设置了 flex-wrap 属性，当 flex-item 的宽度之和（也可以是高度，如果 flex-direction 设置为了 column，这里为了说明都以宽度为例）超过 flex-container 的宽度时，各个 flex-item 会被压缩以将他们放到同一行中，下面是一个例子：

<script async src="//jsfiddle.net/jiananshi/hhegb9cy/2/embed/html,css,result/"></script>

为了避免 flex-item 被压缩，我们可以使用 `flex-shrink` 属性，这个属性的默认值为 1，前面提到了在宽度不足时 flex-item 会被压缩，至于每个 flex-item 压缩多少就是由 flex-shrink 决定的，因此默认每个 flex-item 按照 1:1 来进行压缩，通过将这个属性设置为 0 可以避免元素被压缩。

## flex-basis、width 和 flex-grow 优先级
Flex-basis 这个属性很少见，在看完文档后我也一度觉得这是一个很难掌握的属性，但是查阅 [W3C 规范](https://www.w3.org/TR/css-flexbox-1/#flex-basis-property) 中有这样一句话：

> For all values other than `auto` and `content` (defined above), flex-basis is resolved the same way as width in horizontal writing modes

也就是说除了 `auto` 和 `content` 值之外，他跟 width 表现是一致的，同时 flex-basis 和 width 同时存在时，flex-basis 优先级更高。下面是一个对比这三个属性优先级的例子，可以通过注释掉某个属性更具体的观察两两之间的对比：

<script async src="//jsfiddle.net/jiananshi/nzh2d9bb/embed/html,css,result/"></script>

## inline-flex
创建一个 FFC 除了使用 `display: flex` 之外还可以使用 `display: inline-flex`，正如其名他可以让一个 flex-container 以 inline 的方式渲染，在编写某些 UI 组件时还是挺好用的。