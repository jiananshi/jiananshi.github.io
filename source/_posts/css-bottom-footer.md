---
title: 不定高度内容固定底部 footer
date: 2017-01-18 09:59:01
tags: css
---

这周基本都在写页面，某天遇到一个很常见的需求：一个在每个页面都要用到的 footer，无论页面内容是否超出 100% 都保持在底部。

以前遇到这种场景通常会用 `{position:fixed; bottom: 0}` 来解决，但是到了页面超出 100% 的时候就会挡住底部的内容，后来觉得如果想做好一点不用 JS 不行了。今天不甘心查了查资料发现有种很优雅的纯 CSS 解决方案：

- body 上设置 `{ position: relative; min-height: 100%; }`
这一条是为了让内容高度撑不满时也至少为 100%，当然 `html` 也要设置 `{ height: 100%; }`
- footer 上 CSS 如下：
```css
.footerbottom {
	position: absolute;
	bottom: 0;
	left: 0;
	right: 0;
}
```

将 footer 压到页面底部，同时 `body` 上不要忘记用 `padding-bottom` 为 `footer` 遮盖的部分腾出地方。

最终效果可以看 [DEMO](https://codepen.io/cbracco/pen/zekgx)