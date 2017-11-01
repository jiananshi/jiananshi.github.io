---
title: 使用 webpack.DllReferencePlugin 优化 webpack 打包速度
date: 2016-10-17 19:08:48
tags:
---

最近在写一个 CMS 项目，代码量不大但是每次启动 webpack 打包的时间（注意是首次打包，watch 的更新是不需要整个项目重新打包的）居然需要 14s+，想起去年去一个部门帮忙的时候有个项目每次打包要近 2min（Java 开发一定不会觉得有什么吧），由于项目本身和开发少（其实就是懒），当时也没去解决。遇到了喷一下完事，然而每次赶上发布还是经常被拖后腿 ╮(╯▽╰)╭

## 介绍
这里我们来看一下如何用 webpack 原生提供的 DllReferencePlugin 来加速 webpack 打包

首先介绍一下 dll，用过 windows 的同学应该不陌生，系统或应用配置文件中常常可以看到 .dll 结尾的文件。dll 全称是：dynamic link library，即动态链接库。微软官方有一个很好的例子介绍 dll 的使用场景，windows 系统提供一个 dialog box 库，任何应用都可以直接引用。

将这一思想运用到前端打包，当我们写下诸如 `import 'vue'; import 'angular'; import 'react'` 类似的代码时，webpack 在编译时也会寻找并把它们打包出来，通过将这类变动不频繁的包分离出来可以显著提高 webpack build 的速度（另外，可以将多个项目相同的技术栈抽取出同一个配置文件，共同引用这一个文件）。

## 如何使用 webpack.DllReferencePlugin

首先，我们创建一个 `dll.config.js` 的文件用来配置 dll 文件

```javascript
  const webpack = require('webpack');
  
  // dll lib 的名称
  var name = '[name]_[chunkhash]';

  module.exports = {
    output: {
      path: './dist',
      filename: '[name].js',
      library: name // 这里跟 webpack.DllPlugin 里的 name 保持一致
    },
    entry: {
      vendor: [ 'vue', 'vue-router', 'vue-resource' ]
    },
    plugins: [
      new webpack.DllPlugin({
        path: 'manifest.json',
        name: name, // 这里跟 output.library 保持一致
        context: __dirname // 这里和后面我们要配置的 webpack.DllReferencePlugin 里的 context 保持一致
      })
    ]
  };
```

完成后运行 `webpack --config dll.config.js` 就会在根目录生成 manifest.json 文件和 dist/vendor_mtrwsd219.js（vendor 打包出来的文件）

webpack.config.js 也要配置 manifest 让业务代码中的 `require` `import` 知道如何去寻找 dll 文件中的依赖

```javascript
const webpack = require('webpack');

module.exports = {
  plugins: [
    new webpack.DllReferencePlugin({
      context: __dirname, // 这里要和 dll.config.js 中 webpack.DllPlugin 配置的 context 一致
      manifest: 'manifest.json'
    })
  ]
};
```

配置完成后运行 `webpack` 会发现最终打包出来的文件由于没有去打包 dll 中的文件，文件体积显著减小，webpack 似乎做了缓存，所以我分别在使用 dll 和没有使用的情况下测试了三次打包速度

可以看到在运行次数较少的情况下，配置了 dll 的打包速度提升效果显著，webpack 缓存也很给力，多次运行后两者差距在 1s 左右，目前没有在更大的项目中测试，理论上效果应该会更明显

## 参考

[what is a dll](https://support.microsoft.com/en-us/kb/815065)
[webpack多页应用架构系列（十一）：预打包Dll，实现webpack音速编译(https://segmentfault.com/a/1190000007104372)
