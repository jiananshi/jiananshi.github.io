---
title: 如何在 React 中绑定一个事件处理函数
date: 2018-07-29 18:17:00
tags:
  - javascript
  - react
---

这篇文章的起因是最近看到同事的在处理一个可以响应用户交互需求时的代码：

```js
import React from 'react';

class App extends React.Component {
  render() {
    return (    
    <button onClick={() => alert('Clicked')}>点击</button>
    );
  }
};

render(<App />, document.querySelector('#app'));
```

这段代码的功能很常见，就是在用户点击 `button` 的时候给予对应的交互，在部署到容器中时发现多交互几次之后应用会越来越慢，查看硬件监控会发现内存占用愈来愈高，一位更有经验的同事指出 `onClick` 这里使用匿名函数会导致性能问题，定义到实例上就可以了（得益于 [babel-plugin-transform-class-properties · Babel](https://babeljs.io/docs/en/babel-plugin-transform-class-properties/) 我们可以很方便的这么做）：

```js
import React from 'react';

class App extends React.Component {
  onClick = () => alert('Clicked');
  render() {
    return (    
    <button onClick={this.onClick}>点击</button>
    );
  }
};

render(<App />, document.querySelector('#app'));
```

同事试过之后说问题解决了，至此问题解决，我们可以盖棺定论在 React prop 使用匿名函数会有性能问题 —— 明显很缺乏说服力啊 😂

首先我们查看 babel 编译出来的代码（这里用 babel playground 做了一个最小可复现案例）：

```js
// 编译前
class App {
  onClick = () => alert('Clicked');
}

// 编译后
"use strict";

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var App = function App() {
  _classCallCheck(this, App);

  this.onClick = function () {};
};
```

可以看到 onClick 事件依然在每次实例化时被定义到了实例上，函数创建过多导致性能问题是不成立的，从「减少函数创建次数」这个角度我们想要进行优化，只要将函数的 this 绑定到基类上即可，我们可以借助 bind 完成这件事：

```js
import React from 'react';

class App extends React.Component {
  constructor() {
    super();
    this.onClick = this.onClick.bind(this);
  }
  onClick() { alert('Clicked'); }
  render() {
    return (    
    <button onClick={this.onClick}>点击</button>
    );
  }
};

render(<App />, document.querySelector('#app'));
```

盗了 [dailyjs]([Demystifying Memory Usage using ES6 React Classes – DailyJS – Mediumhttps://medium.com/dailyjs/demystifying-memory-usage-using-es6-react-classes-d9d904bc4557) 上两张图表示这种方式是如何节省内存的（虚线框代表从父类继承的方法，实线框代表占用内存的部分）：

![1](//pic.yupoo.com/jiananshi/8c2db9bc/16b5908c.png)

![2](//pic.yupoo.com/jiananshi/0f49b19d/45f41c33.png)

每次都要在初始化的时候写一坨 `bind` 确实很恶心，可以参考 autobind 这个 decorator 用 babel 转一波 ，在 React 中绑定一个事件处理函数的终极方案是：

```js
import React from 'react';

class App extends React.Component {
  @autobind
  onClick() { alert('Clicked'); }

  render() {
    return (    
    <button onClick={this.onClick}>点击</button>
    );
  }
};

render(<App />, document.querySelector('#app'));
```

最后，匿名函数在某些场景下确实会导致性能问题，就是当配合 PureComponent 使用时，在 Prop diff 的时候做 shallowEqual 的结果总会是 false，因此即使 state/props 没有变动也会进入后面的阶段。
