---
title: 开始使用 Webp 图片
date: 2017-01-06 22:40:06
tags:
---

![webp](http://oiw32lugp.qnssl.com/2017-01-06-144053.jpg?imageMogr2/format/webp)

Webp 是一种图片格式，记得刚进饿厂的时候 gogu 就在主站推了，当时没重视（其实当年 gogu 就在搞 Reactive 和 FP，当然这都是后来回头看了，其中也有没成的 😂  ）

Webp（发音 webpy）是谷歌创造的一种是支持有损压缩和无损压缩的图片文件格式，无损压缩后的 Webp 比 png 文件少了 26％ 的体积，有损压缩后的WebP图片相比于等效质量指标的JPEG图片减少了 25％~34% 的体积。

对应的，Webp 的代价是：与JPG相比较，编码速度慢 10 倍，解码速度慢 1.5 倍。但是由于体积上显著的减小，即使在轻量级的业务中 Webp 最终性能还是优于普通 jpg/png。

前端工程师看到这里理所应当会顾虑 Webp 的兼容性，下面是 caniuse 上的截图：

![webp caniuse](http://oiw32lugp.qnssl.com/2017-01-06-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-06%20%E4%B8%8B%E5%8D%8810.50.20.png)

可以看到在 Chrome/Android/opera 上对于 Webp 的支持很好（苹果为啥不滋瓷下呢 😣  )，所以如果你的网站想要使用 Webp，强烈建议你添加一段检测是否支持 Webp 的代码：

```javascript
'use strict'

var image = new Image();
image.onload = () => { /* support webp */ };
image.onerror = () => { /* not support webp */ }
image.src = 'http://yoursite.com/image.webp' // 可以放一个 1x1 的图片
document.body.appendChild(image);
```

值得一提的是，七牛的图片处理接口也支持 Webp 格式转换了：`yourimage?imageMogr2/format/webp`，如果你能看到本文开头 Webp 的 Logo，说明你的浏览器已经可以渲染 Webp 了。


后记，博客的主题太尼玛丑了一直按下不表小修小补，周末准备自己撸袖子搞一波了。

