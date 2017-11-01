---
title: Margin-collapsing in CSS
date: 2016-10-12 08:35:09
tags:
---

写页面的时候经常会遇到一种情况，子元素垂直方向上的 Margin 经常会连带父元素一起往对应方向移动，写了几年页面，虽然也知道 Margin Collapsing 的概念，但每次遇到这种情况都是偷懒用 padding 了事 ╮(╯▽╰)╭

很多时候还是要放下一些东西，像 gogu 说的 「笨拙的做自己」，那么来吧，直接看 W3C 规范里是怎么讲的：

以下几种情况不会发生 Margin 合并：

## 不会发生 Margin 合并

- root 级别的 element （这里指的就是 html 元素）
- 一个 element 上下 margin 相邻（即 height 为 0），并有 clear 属性（这么说只是方便理解，原文为：with clearance），它同之后的兄弟元素 margin 合并但合并后的 margin 并不会同父元素的 margin-bottom 合并（demo: https://gist.github.com/anonymous/015d84ed1e9f602a9a0654e8c3ddb0f8t）
- 水平方向上的 margin

## Margin 相邻（adjoining）

- 都是 normal-flow 中的 block 级别元素
- 两者中间没有 line box / clearance / padding / border
- 都处于垂直方向上相邻的 box 边缘，有以下几种情况：

1. box 的 margin-top 和它第一个在 normal flow 里的子元素的 margin-top
2. box 的 margin-bottom 和它后面的兄弟元素，这个兄弟元素也要在 normal-flow 中
3. normal flow 中最后一个元素的 margin-bottom 和它的父元素的 margin-bottom（注：父元素的高度必须是自动计算出来的）
4. 没有创建新的 BFC / `min-height` 计算出来为 0/ `height` 为 0 或者 `auto` / normal-flow 中没有子元素，满足以上几个条件的 box 的上下 margin 将会合并

（注：margin-adjoining 也可以在非兄弟/子元素之间发生）

## Margin 合并规则

当多个 Margin 合并时，都为正数时取最大的作为最终 margin；有负数的时候从最终计算的值中扣掉负数；都是负数的时候取绝对值大的作为最终 margin

## Addition

- 浮动元素不和其他元素 margin-collapse（和浮动元素之间也不会）
- 产生 BFC 的元素不和 normal-flow 中的子元素产生 margin-collapse
- 绝对定位元素不和其他元素 margin-collapse
- inline-block 元素不和其他元素 margin-collapse
- block 元素的 margin-bottom 总是同它之后处于 normal-flow 中的 block 元素的 margin-top 合并（除非它有 clearance）
- normal-flow 中的 block 元素的 margin-top 同它第一个 normal-flow 中的 block 子元素的 margin-top 合并，父元素不能有 border-top / padding-top，子元素要没有 clearance
- normal-flow 中的 block 元素的 margin-bottom 同它最后一个 normal-flow 中的 block 子元素的 margin-bottom 合并，父元素 height 为 `auto` / min-height 为 0 / 没有 border-bottom / 没有 padding-bottom，子元素的 margin-bottom 没有同其他被 clearance 的元素的 margin-top 合并


妈的以后还是用翻译或阅读源码的形式来写这种题材吧，后面也会补一些 demo，写的太痛苦了。
另外关于 BFC、clearance、normal-flow 其实还是很大的话题，后面有机会会写一下

参考：

- [W3C](https://www.w3.org/TR/CSS2/box.html#collapsing-margins)