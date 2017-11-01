---
title: 记一次用 MutationObserver debug 的经历
date: 2017-03-22 23:13:05
tags:
  - mutation-observer
  - javascript
  - dom
---

> 上篇文章在创建 Microtask 中提到了 MutationObserver 的回调函数会创建一个 microtask，很巧的是这周刚好用上了这个 API 并且快速的解决了一个很诡异的问题。

有天下午上班间隙被叫去看一个问题，一个 Node 节点在添加新的 Node 之后它的子节点依然是空的，在检查了拼写、时序等问题后还是没有头绪，这时获取到一个信息：“如果把 DOM 插入操作放到 setTimeout 里执行就可以”，我立刻想到可能是因为插入时 DOM 还没有创建，于是试着在控制台打印，结果是有的… 最后我试着在被插入的 DOM ParentNode 里放了一点内容，刷新之后内容没了，至此我才开始怀疑是有其他脚本在本次 task 中清空了 Node 节点，因此 setTimeout 创建的新的 Task 幸免于难。

回过头来重新审视这个页面，里面充斥着各种 jq 插件，debug 几乎是一项不可能短时间内完成的任务，接下来的问题就很简单了：如何证明这个 DOM 被修改过？我首先想到为这个 dom 创建一个 proxy，于是我试着将 Node 对象改成一个 Proxy：

![screencast](http://oiw32lugp.qnssl.com/2017-03-22-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-22%20%E4%B8%8B%E5%8D%8811.31.14.png)

如图所示，新创建的 a 标签并没有被 proxy，如果怀疑 a 并不是 Node 的实例的话可以试着运行：

```javascript
> a instanceof Node
true
```

当然，这里的 Node 是 proxy 之前的 Node，具体为何会这样我也还没想清楚（作为下一篇博客的素材，如果你知道还请在评论里指导我），总之目前来说这个方案破产了。

这时我突然想起来之前可以为 Node 节点添加观察者的对象：MutationObserver

## MutationObserver
MutationObserver 是一个全局的构造函数，为了观察目标 DOM，我们首先需要创建一个 observer 实例：

```javascript
const observer = new MutationObserver(mutations => {
  mutations.forEach(console.log.bind(console));
});
```

MutationObserver 接收一个回调函数，回调函数的参数是一个 `MutationRecord<List>`，下面列举一下 MutationRecord 对象的结构：

| 属性 | 类型 | 描述 |
| ---- | ----| ---- |
| type | String | 根据 DOM 变动情况返回 attributes/characterData/childList |
| target | Node | 返回此次变化影响到的节点，具体返回那种节点类型是根据 type 值的不同而不同的。如果 type 为 attributes，则返回发生变化的属性节点所在的元素节点,如果 type 值为 characterData，则返回发生变化的这个 characterData 节点。如果 type 为 childList，则返回发生变化的子节点的父节点。|
| addedNodes | NodeList | 返回被添加的节点 |
| removedNodes | NodeList | 返回被删除的节点 |
| previousSibling | Node | 返回被添加或被删除的节点的前一个兄弟节点 |
| nextSibling | Node | 返回被添加或被删除的节点的后一个兄弟节点 |
| attributeName | String | 返回变更属性的名称 |
| attributeNamespace | String | 返回变更属性的命名空间 |
| oldValue | String | 根据 type 值的不同，返回的值也会不同。如果 type 为 attributes，则返回该属性变化之前的属性值。如果 type 为 characterData，则返回该节点变化之前的文本数据。如果 type 为childList。则返回 null |

接下来我们需要为观察者 `observer` 指定观察的对象以及观察的类型，MutationObserver 支持如下几种类型：

| 属性 | 描述 |
| ---- | --- |
| childList | 需要观察目标节点的子节点 |
| attributes | 需要观察目标节点的属性节点 |
| characterData | 节点的文本内容是否发生变化 |
| subtree | 除了目标节点，如果还需要观察目标节点的所有后代节点（包括上述三种属性变化） |
| attributeOldValue | 将发生变化的属性节点之前的属性值记录下来（会记录到 MutationRecord.oldValue），需要注意的是这个属性开启需要将 attributes 属性开启的前提 |
| characterDataOldValue | 将发生变化的characterData节点之前的文本内容记录下来（同样会被记录到 MutationRecord.oldValue）需要开启 characterData 属性 |
| attributeFilter | 指定观察的属性名（比如：[‘className’, ‘style’] |

在前面提到的情况中，我们需要观察 DOM 子节点的变化，开启 childList 就够用了：

```javascript
const $target = document.querySelector('#ymy');
new MutationObserver(console.log.bind(console))
  .observe($target, {
    childList: true
  });
```

之后刷新浏览器观察 Observer 的输出：

![screencast](http://oiw32lugp.qnssl.com/2017-03-22-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-23%20%E4%B8%8A%E5%8D%8812.22.35.png)

可以看到 removedNodes 属性中有一条记录，明确表示这个 DOM 节点的子节点被移除过，后面的事就是开发自己慢慢 debug 了。

## 后记
问题是解决了，但是我产生一个困惑，为什么 observer 的回调参数会是一个数组？看了文档才知道 DOM 的变动不会立刻触发 observer 回调，而是等本次 task 中所有的变动都生效后才运行。举个例子，在一个 task 中对一个 ul 节点插入 10000 个子节点，那么 observer 的回调也仅仅只会运行一次。想到这里，我基本已经确定 observer 的回调函数也是一个 microtask 了。

为了证实我的猜想，在 Chrome 控制台写了如下代码测试：

```javascript
var a = document.querySelector('a');
// microtask 1
Promise.resolve().then(() => console.log(2));
new MutationObserver(console.log.bind(console))
  .observe(a, {
    childList: true
  });
// task
setTimeout(() => a.innerHTML = 'ymy', 0)
a.innerHTML = ''
// microtask 2
Promise.resolve().then(() => console.log(3))
a.innerHTML = 'ymy';
a.innerHTML = '';
console.log(1);
```

输出如下：

```bash
> 1
> 2
> [MutationRecord, MutationRecord, MutationRecord]
> 3
> [MutationRecord]
```

这段代码的执行顺序如下：

1. setTimeout 将一次 dom 变动插入下一个 task 队列
2. DOM 变动执行完毕
3. 本次 task 结束前，运行 `console.log(1)`
4. 开始处理 microtask，首先是 microtask1 `console.log(2)`
5. 处理 observer 回调，输出本地 DOM 变动影响的 `childList`
6. 处理 microtask2 `console.log(3)`
7. 本次 task 结束
8. 下一个事件队列结束前处理 microtask，observer 回调再次运行
