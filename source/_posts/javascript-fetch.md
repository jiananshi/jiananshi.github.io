---
title: 「译」That's so fetch!
date: 2017-03-28 03:29:56
tags:
  - javascript
  - fetch
  - stream
---


> 记得去年看到过 @sodatea 跟人讨论时说：「Fetch 并不仅仅是 XHR 包了一层 Promise 而已」，我在后面很长一段时间里一直不解其意，直到后来对 Stream 有所了解并阅读完这篇文章后才体会到个中精妙。Stream 是我非常希望学习掌握的技术之一，不出意外后面几篇文章都会是关于 Stream（流）的。
> 另外博客换了新主题，被某人说不如之前好看，果然美感是需要培养的呀 ╮（╯＿╰）╭

我注意到大家对于 Fetch 的 [API 设计](https://fetch.spec.whatwg.org/#fetch-api) 有一些疑惑，是时候写一篇文章来说清楚这件事了。

首先你会发现在 API 设计上 Fetch 比 `XMLHttpRequest` 要好太多了，下面是如何通过 XHR 获取 JSON 数据：

```javascript
var xhr = new XMLHttpRequest();
xhr.open('GET', url);
xhr.responseType = 'json';

xhr.onload = function() {
  console.log(xhr.response);
};

xhr.onerror = function() {
  console.log("Booo");
};

xhr.send();
```

下面让我们看一下如何用 fetch 做同样的事：

```javascript
fetch(url).then(function(response) {
  return response.json();
}).then(function(data) {
  console.log(data);
}).catch(function() {
  console.log("Booo");
});
```

替换成 ES6 的箭头函数语法会让代码看起来更简洁：

```javascript
fetch(url).then(r => r.json())
  .then(data => console.log(data))
  .catch(e => console.log("Booo"))
```

再引入 [ES7 async 语法](https://jakearchibald.com/2014/es7-async-functions/) 可以像写同步代码一样写异步代码：

```javascript
(async() => {
  try {
    var response = await fetch(url);
    var data = await response.json();
    console.log(data);
  } catch (e) {
    console.log("Booo")
  }
})();
```

不幸的是，并非所有人都认同 Fetch 的设计，某个热门的 Javascript 社区成员拒绝接受...

![screencast](http://oiw32lugp.qnssl.com/2017-03-26-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-26%20%E4%B8%8A%E5%8D%8811.56.59.png)

他认为我们应当停止继续增加高阶 API，特别是目前的形势是在请求和响应上我们越来越需要支持一些底层的操作。

对于这点我只能说，Fetch 就是如此，下面让我们来澄清这些误会吧...

### 不要被 Fetch 优雅的 API 所迷惑
一个优雅简洁的 API 可能是高阶 API 的一个特征，但这并不是 Fetch，XHR 设计的一团糟，一个更低层更多功能的 API 也可以设计的简单易用。

目前 XHR 的定义依赖于 Fetch（查看 [XHR `send` 方法的调用](https://xhr.spec.whatwg.org/#the-send%28%29-method)），这意味着 Fetch 比 XHR 更接近底层。

### Fetch 还在不断完善
今天在 Chrome 中 fetch 的功能还没有完全实现文档中的规定，而文档也没有覆盖所有计划中的功能，这种情况有时候是因为压根没有开始设计，也有的是因为它们依赖其他标准。标准的形成基本如下图：

![spec](http://oiw32lugp.qnssl.com/2017-03-26-040458.jpg)

有件事很困扰我，作为开发者我们提倡不断迭代、发布，而当我们作为使用者的时候，对这一策略的反应往往是：**你怎么好意思告诉我这玩意是残缺的？**。

另一个方式是为了一个功能大家一起坐下来花上几个月（或者几年？）讨论，而不是像现在这样大部分由不同的开发者来完成，迭代开发同样意味着我们可以不断的接收到用户的反馈，这些意见也许会影响到我们后面的设计和优先级。

好了，现在让我们看一下 Fetch 能做什么 XHR 不能做的事：

## 原生的 Request/Response
可以说 XHR 把 request 和 response 混在一起了，这意味它们没有被区别开。多亏了 `Request` 和 `Response` 构造函数，Fetch 可以区别两者，这一点在使用 ServiceWorker 的时候很有用：

```js
self.addEventListener('fetch', function(event) {
  if (event.request.url === new URL('/', location).href) {
    event.respondWith(
      new Response("<h1>Hello!</h1>", {
        headers: {'Content-Type': 'text/html'}
      })
    )
  }
});
```

在上面的代码中，`event.request` 就是一个 `Request` 对象，这里没有产生 response，我用自己构造的 `Response` 实例返回给了客户端，避免了这次网络请求。或者，我可以通过 `fetch(event.request)` 获取网络请求，甚至从 cache 里返回 response。

[Cache API](https://slightlyoff.github.io/ServiceWorker/spec/service_worker/#cache-objects) 是一组把 `Request`、 `Response` 用作键值对的存储结构，并且允许开发者为它们分组。

这段代码是从最新版本的 Chrome 中 ServiceWorker 脚本中截取的，在 Chrome Beta 也允许使用 Fetch API。

不久之后你就可以通过 `request.context` 获取请求的来源，这样开发者就可以区分通过超级链接点击发起的请求和 `<img>` 标签发出的请求，等等。

## ‘no-cors’ 和不透明的响应
如果我在这个页面通过 XHR 请求 `//google.com` 或者简单的 fetch 一定会失败，因为这是一个 CORS 请求并且响应没有 CORS headers。

不过有了 fetch 你可以发一个 `no-cors` 请求：

```js
fetch('//google.com', {
  mode: 'no-cors'
}).then(function(response) {
  console.log(response.type); // "opaque"
});
```

这个请求类似 `<img>` 标签发出的请求，当然，你不能获得 Response 的内容，因为里面可能有私有的信息，但它可以被别的 API 消费：

```js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    fetch('//www.google.co.uk/images/srpr/logo11w.png', {
      mode: 'no-cors'
    })
  )
})
```

上面的这段代码在 ServiceWorker 中可以正常运行，只要接收方可以接收一个 `no-cors` 响应，比如 `<img>`，而 `<img crossorigin>` 不行。

你也可以通过 Cache API 缓存这些响应，这对于一些存储在 CDN 上的脚本、CSS、图片等等通常没有设置 CORS headers 的资源很合适。

## Streams（流）
XHR 没有流的功能，虽然你可以通过 `.responseText` 在响应尚未结束时获取响应，但这会导致整个响应在内存中被缓存。

通过 fetch 你可以获取底层的请求体流，假设我们现在想要加载一个很大的 CSV 并且找到内容是 “Jake” 的单元格：

```js
fetch('/big-data.csv').then(function(response) {
  var reader = response.body.getReader();
  var partialCell = '';
  var returnNextCell = false;
  var returnCellAfter = "Jake";
  var decoder = new TextDecoder();

  function search() {
    return reader.read().then(function(result) {
      partialCell += decoder.decode(result.value || new Uint8Array, {
        stream: !result.done
      });

      // 把内容分割成 CSV 单元格
      var cellBoundry = /(?:,|\r\n)/;
      var completeCells = partialCell.split(cellBoundry);

      if (!result.done) {
        // 上一个单元格可能还没有读完
        partialCell = completeCells[completeCells.length - 1];
        // 从已读完的单元格数组中移除
        completeCells = completeCells.slice(0, -1);
      }

      for (var cell of completeCells) {
        cell = cell.trim();

        if (returnNextCell) {
          reader.cancel("No more reading needed.");
          return cell;
        }
        if (cell === returnCellAfter) {
          returnNextCell = true;
        }
      }

      if (result.done) {
        throw Error("Could not find value after " + returnCellAfter);
      }

      return search();
    })
  }

  return search();
}).then(function(result) {
  console.log("Got the result! It's '" + result + "'");
}).catch(function(err) {
  console.log(err.message);
});
```

这里我每次通过往内存中放一小部分 CSV 文件的内容来读取它，当我找到我需要的内容时就取消掉 stream，关闭连接。

按照 [streams 标准](https://streams.spec.whatwg.org/)，`response.body` 是一个 `ReadableStream`，我们从一开始就计划使用 Stream。

`TextDecoder` 是 [编码标准](https://encoding.spec.whatwg.org/#interface-textencoder) 的一部分，如果 `decode(input, {stream: true})` 收到的数据块是一个多字节数据的一部分（译者注：也就是发生了截断），它会返回并清除这部分数据之外的内容，下一次 decode 调用会基于截断的部分继续。

在 Chrome Canary 中已经可以实现这些功能了，关于我前面说的东西 [这里有一个 DEMO ](https://jsbin.com/gowaze/quiet) ，还有一个 [DEMO](https://domenic.github.io/streams-demo/) 附有一个很大的数据集（注意，运行这个 demo 可能会下载几兆的数据）。

Stream 是我非常希望在 web 上看到的技术之一，我希望可以用 Stream 去读取一些 JSON 数据，根据它创建 HTML 片段最后再通过 Stream 返回给浏览器，基于 JS 的应用缺乏一种从一个数据源渐进式加载渲染的方式，stream 可以解决这个问题。

Transform stream 马上就会推出了，有了它可以进一步简化上面的代码。理想的情况是：`TextDecoder` 是一个 transform 流，另一个 transform 流可以把它的输出转为 CSV 文件行。

```js
fetch('/big-data.csv').then(function(response) {
  var csvStream = response.body
    .pipeThrough(new TextDecoder)
    .pipeThrough(new CSVDecoder);

  csvStream.read().then(function(result) {
    // 第一行 CSV 文件的数组
    console.log(result.value); 
  });
});
```

Transform stream 在和 ServiceWorker 共同使用的时候也很有趣：

```js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    fetch('video.unknowncodec').then(function(response) {
      var h264Stream = response.body
        .pipeThrough(codecDecoder)
        .pipeThrough(h264Encoder);

      return new Response(h264Stream, {
        headers: {'Content-type': 'video/h264'}
      });
    })
  );
});
```

上面的代码中我使用 transform stream 将浏览器无法识别的视频资源用 JS 解码然后在编码成浏览器可以识别的格式，如果浏览器可以实时完成这样的事会很 cool。

## Stream 读取器和克隆
正如我前面所说，为了让开发者可以立刻享受到其他特性，我们暂时为 fetch 去掉了 Stream 的支持，为了弥补 stream 的缺乏和提供一种数据读取方式，我们提供了一些阅读器作为替代：

```js
fetch(url).then(function(response) {
  return response.json();
});
```

这个 reader 会将整个 stream 当作 JSON 一样读取，下面是完整的阅读器列表：

- `.arrayBuffer()`
- `.blob()`
- `.formData()`
- `. json()`
- `.text()`

在 `Request` 对象上也有这些方法，举个例子，你可以用它们在 ServiceWorker 里读取 POST 请求的数据。

它们是 stream 读取器，这意味着它们也符合流的特性：

```js
fetch(url).then(function(response) {
  return response.json().catch(function() {
    // 这段代码不会起作用的
    return response.text();
  });
});
```

`.text()` 方法的调用失败了，因为 stream 已经被消耗了，你可以通过 `.clone()` 方法解决这个问题：

```js
fetch(url).then(function(response) {
  return response.clone().json().catch(function() {
    return response.text();
  });
});
```

`.clone()` 方法会对 Response 进行缓存，clone 结果会被以 JSON 的方式读取、消耗，而原始的 Response 不会受到任何影响。当然，这意味着原始的 Response 数据会一直保留在内存中直到所有的副本被消耗或 GC。

你也可以读取 response 的 header：

```js
fetch(url).then(function(response) {
  if (response.headers.get('Content-Type') === 'application/json') {
    return response.json();
  }
  return response.text();
});
```

这是 fetch 比 XHR 好的另一个点，你可以在读取 header 之后决定用哪种格式去读取响应内容。

## 其他相关
还有很多 fetch 优于 XHR 的地方我不准备一一展开说明，这包括：

- Headers 类
Fetch 有一个 [headers 类](https://fetch.spec.whatwg.org/#headers-class) 可以用来读写 header，它同时是一个 ES6 迭代器
- 缓存控制
[cache mode](https://fetch.spec.whatwg.org/#concept-request-cache-mode) 让开发者可以决定如何同缓存通信，比如是否应该查询缓存？如果缓存没有过期，响应是否应该直接使用缓存？响应可不可以只从缓存中取？
后者是一个有争议的话题，因为它涉及到了用户隐私，Chrome 最终可能会通过 CORS 来限制它。
- No-credential 的同源请求
XHR 要求同源请求必须是 `with-credentials` 的，而在 Fetch 中这不是必须的。

## 目前还有什么缺陷？
当然，有一些 XHR 的特性 fetch 还没有。

### 抛弃一个请求
这是一个很大的问题，Canary 版本允许你取消 Stream，但是没有任何办法可以在响应 header 读取前抛弃一个请求。

我们计划通过可取消的 promise 来解决这个问题，很多其他的标准也依赖这个，对此你可以查看 [Github 上的讨论](https://github.com/slightlyoff/ServiceWorker/issues/625)。

### 进度事件
Progress 事件是 fetch 暂时不会开放的高级特性，你可以通过使用 pass-through stream 根据 Content-Length 来模拟。

这意味着你无法处理没有 Content-Length 头的响应，当然，即使有它也可能是个假 Content-Length，但是使用 Stream 就可以不依赖它的准确性了。

### 同步请求
同步请求不是个好的设计，Fetch 不会实现它。（译者注：某些场景还是需要的，比如支付宝支付跳转新页面，`open` 是不能在事件 stack 之外异步调用的，这就需要使用原生 XHR 或者更改业务流程了）。

## 这些不能基于 XHR 封装一层吗？
XHR 的设计很糟糕，现在已经是 2016 年了，当我们意识到 XHR 很难用的时候已经吃了足够多的苦头了。

我承认，这其中很多特性可以基于 XHR 二次封装，但这看起来更像是在 hack，Fetch 让我们可以抛开 XHR 设计糟糕的 API，添加底层功能并提供更好的 API 接口，同时我们可以使用现代 JavaScript 特性比如 promise 和 iterators。

如果你想从现在开始停止使用 XHR，推荐你试试 [fetch polyfill](https://github.com/github/fetch)（译者注：现在是 2017 年了，在最新的 Safari 里还是要用这个 :joy:），它内部是基于 XHR，所以它不能实现 XHR 没有的功能，但你可以通过它享受 fetch 的 API。

## 进一步阅读
- [Intro to ServiceWorker](http://www.html5rocks.com/en/tutorials/service-worker/introduction/)
- [ES6 iterators]( [http://www.html5rocks.com/en/tutorials/service-worker/introduction/)
- [ES7 async functions](https://jakearchibald.com/2014/es7-async-functions/)
- [Partial fetch polyfill](https://github.com/github/fetch) - 基于 XHR



