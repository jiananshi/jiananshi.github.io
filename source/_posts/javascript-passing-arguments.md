---
title: JS 中参数的传递方式
date: 2017-04-02 10:42:44
tags: javascript
---

提到 Javascript 参数的传递方式，大部分稍微有点经验的人都能说出有按值和按引用传递，基础类型数据按值传递，Object 按引用传递，比如下面的代码：

```js
function parse(raw) {
  raw = 10;
}

var num = 5;
parse(num);
num // 5
```

由于基础类型按值传递，所以函数 `parse` 并不会改变外面的 `num` 变量的值，仅仅是创建了一个局部变量 `raw` 并且设置为 10，这个变量由于没有被引用后面很快就会被 GC 了。

接下来就是有趣的部分了：

```js
function parse(raw) {
  raw.name = 'ymy';
}

var obj = { name: '' };
parse(obj);
obj.name // 'ymy'
```

```js
function parse(raw) {
  raw = { name: 'ymy' };
}

var obj = { name: '' };
parse(obj);
obj.name
```

这里可能会有人觉得既然 JS 里对象的传递是按引用，那么函数会改变 obj 引用的地址，obj.name 的输出结果应该是 `ymy`，但这里实际上输出的是 `''`。

这里涉及到一种参数的传递方式，叫做 `call by sharing`，函数 `parse` 实际上接受到了 obj 引用的一个副本，直接修改 raw 会导致它指向另一个地址，所以并不会影响到外层的 `obj`。看网上说不仅 JS(ECMAScript)，Java、Python 和 Ruby 也有这样的特性，个人感觉这个概念有点像「指向指针的指针」，有懂低层实现的麻烦在评论里指导下。

想起以前问别人的时候这个还算个必问环节，其实自己理解也不深刻，以后提到 JS 的参数传递方式可不能简单的只说按值和按引用了。