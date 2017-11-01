---
title: NPM 管理依赖与版本
date: 2016-12-28 03:37:08
tags:
---

## NPM 依赖管理

NPM 通过 `package.json` 文件进行记录依赖的包，通常我们的依赖被划分为三类：

- dependency
- devDependency
- peerDependency

假设我们在用 `React` 技术栈写业务，那么类似 `react`、`react-router` 和 `redux` 毫无疑问要配置到 dependency 中，不装它们项目肯定是运行不起来的。

devDependency 用来配置测试/开发时需要的包，换句话说用户通过 `npm install` 你的项目后，他并不关心你开发的技术栈和测试环境，只想要 build 出来的文件。它可能是 bin 也可能是一个 js 文件，类似 `webpack`、`karma`、`babel` 等工具不应当被客户端下载。

起初我也很疑惑 peerDependency 是做什么用的，如果你有过开发插件应该可以很快理解，peerDependency 是指宿主提供的依赖，就好比我们开发了一个插件叫做 `vue-popup`，那么理所应当安装这个包的人应当已经安装了 `vue`，我们可以在配置中这样写：

```json
{
  "peerDependencies": {
    "vue": "^2.1.0" 
  }
}
```

## 依赖包版本管理

NPM 包的版本遵从[语义化原则](http://semver.org/lang/zh-CN/)

假设我们有一个包叫做：`my-package`，版本是 1.2.3。其中 1 代表主版本号、2 是次版本号，3 代表修订号。

**通常** 只有在发布了 API 不兼容的版本时更新主版本号，向下兼容的版本则是更新次版本号，bugfix 等更新修订号。在配置依赖时，我们通常不会关注每个依赖包的状态并且及时更新新版本修复 `bug`，这个流程是可以自动化的：

```javascript
{
  "dependencies": {
    "left-pad": "~1.0.0", // 仅更新修订号
    "vue": "^2.1.0", // 最多只更新次版本号
    "angularjs": "*" // 永远安装最新的版本
  }
}
```

我们不能保证所有开发者的包都是严格遵循 semver 的，所以根据实际情况严格指定版本号，或者使用 [shrinkwrap](https://docs.npmjs.com/cli/shrinkwrap) 或者 [yarn](https://yarnpkg.com/) 类似的工具加强对包版本的控制。