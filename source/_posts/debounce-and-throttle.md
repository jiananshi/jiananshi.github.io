---
title: Debounce and Throttle
date: 2016-10-19 09:38:54
tags:
  - javascript
---

# Debounce and Throttle
Debounce 和 Throttle 是两个很相似的可以让我们控制单位时间内函数执行次数的设计，这篇文章主要写一下他们的原理和适用场景。

## Debounce 
![img](http://pic.yupoo.com/jiananshi/ac7553b5/7a3aa49f.png)

Debounce 限制函数在规定的操作间隔时间只运行一次，如果在不到时间的情况下仍接收到执行操作，那么函数不会执行并且重新计时。这么说可能有点绕，结合实际应用场景来理解：我们有一个输入框会根据用户输入实时发送请求获取联想词，我们不希望用户每次敲击键盘都发送请求，理想情况是用户停止输入 Xms 后才发送请求，并且如果在这期间用户继续敲击键盘则重新计时，实现如下：

```js
function debounce(func, delay = 0) {
  let timer, result;

  function execute(context, args) {
    timeout = null;
    if (args) result = func.apply(context, args);
  }

  function debounced(...args) {
    if (timer) clearTimeout(timer);
    let self = this;
    timeout = setTimeout(function() {
      execute(self, args);
    }, delay);
    return result;
  }

  return debounced;
}
```

存在一个问题就是函数不会立刻执行，而是要经过指定时间间隔后才会第一次执行，很多情况下我们不希望这样，做一个优化：

![img](http://pic.yupoo.com/jiananshi/82015895/15b390a2.png)

```js
function debounce(func, delay = 0, immediate) { // 新增 immediate 参数
  let timer, result;

  function execute(context, args) {
    timeout = null;
    if (args) result = func.apply(context, args);
  }

  function debounced(...args) {
    if (timer) clearTimeout(timer);
    if (immediate) { // immediate 为 true 时检查是否可以立即执行
      let callNow = !immediate;
      timeout = setTimeout(execute, delay);
      if (callNow) result = func.apply(context, args);
    } else {
      let self = this;
      timeout = setTimeout(function() {
        execute(self, args);
      }, delay);
    }
    return result;
  }

  return debounced;
}
```

## Throttle
Throttle 的功能是确保函数在 Xms 内只执行一次，同 debounce 不同的是它提供的是一个时间窗口，在窗口期内额外的调用并不会重置计时，throttle 只是单纯的无视了它，实现如下：

```js
function throttle(func, delay) {
  let timer, context, result;
  let previous = 0;

  function execute(...args) {
    previous = Date.now();
    timeout = null;
    result = func.apply(context, args);
  }

  function throttled(args) {
    let now = Date.now();
    let remaining = delay - (now - previous);
    context = this;
    if (remaining <= 0 || remaing > delay) {
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }
      previous = now;
      result = func.apply(context, args);
    } else if (!timer) {
      timeout = setTimeout(function() {
        execute(...args);
      }, remaining);
    }
    
    return result;
  }

  return throttled;
}
```

## Reference
[underscore.js](https://underscorejs.org/docs/underscore.html)
[css-tricks](https://css-tricks.com/debouncing-throttling-explained-examples/)

