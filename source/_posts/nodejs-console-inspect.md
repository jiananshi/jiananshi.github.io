---
title: Console in Nodejs
date: 2017-04-08 00:01:41
tags:
  - nodejs
  - javascript
---

在 JS 中 console 的作用类似其他语言的 print、echo，在开发调试过程中常常会用到。上周在工作中用 console 调试代码的时候遇到了 console 在 Nodejs 中一个神奇的行为，这里记录下。

<!-- more -->

在一段业务逻辑中我拿到了数据库中的一条记录，类似：

```js
var record = {
  id: 1,
  name: 'ymy',
  weight: 49
};
```

接下来我要把这条记录的 key 都提出来做下一步处理：

```js
Object.keys(record).reduce((base, item) => {
  /* do things */
}, Object.create(null))
```

接着我发现在后续处理中 `Object.keys` 提取到的 key 同我用 console 打印出来的不同，而是类似 `$_` 和 `_doc` 这样的 key。我首先想到这个 record 可能是被加特技了，所以通过 `JSON.stringify(JSON.parse(record))` 的方式先尝试了一下，发现拿到的值是我想要的（事后证明这个测试也是错的，record 上面有自定义的 `toJSON` 方法）。

现在我已经有至少两种方式完成这个工作，通过 `_doc` 去获取 record 的值或者用上面提到的 JSON 的方式转，不过那天工作量比较不饱和，我决定花点时间弄清楚为什么会这样。

去翻了 mongoose 的源码发现并没有重写了 valueOf 和 toString 方法，接着我注意到源码里有两个方法**可能**会返回这样的结果：`getValue` 和 `inspect`，这时怀疑的对象才转移到 console 这个类上，去 nodejs 源码里看一波它的实现。

![console source code](http://oiw32lugp.qnssl.com/2017-04-07-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-08%20%E4%B8%8A%E5%8D%8812.26.50.png)

这里截了一段 `console.log` 的实现，write 函数内部实现就是用 stream 往 stdout 写数据，而输出的数据并不是单纯的字符串，而是依赖 `Util` 标准库处理过的内容，问题应该就是在这里了，再去看一波 util.js

![util source code](http://oiw32lugp.qnssl.com/2017-04-07-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-04-08%20%E4%B8%8A%E5%8D%8812.31.35.png)

这段代码大部分是各种过滤和防御，需要注意的是 util 的配置上有一项叫做：`customInspect` 的属性，默认值是 true，它会检查传进来的对象上是否存在 inspect 方法，有的话在输出时就返回自定义的 inspect，基于此我们可以编写如下的代码：

```js
var record = {
  name: 'ymy',
  weight: 49,
  inspect() {
    return {
      name: 'ymy',
      weight: 52
    };
  }
};

console.log(record); // { name: 'ymy', weight: 52 }
```

至此问题解决了，头一次看了点 node 的源码觉得还是挺厉害，在类型、空值和死循环上各种防，通过 `Object.seal` 保护默认配置项（JS 中 const 声明的对象是个假 const），还有大量使用 symbol 做 key 防冲突。了解了这一特性之后觉得自己用的可能是个假 console…


唉，都快写完了，还没等来 wuli 萦儿
