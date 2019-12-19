---
title: webpack原理与实战
date: 2017-05-31T08:11:45Z
url: https://github.com/gwuhaolin/blog/issues/4
tags:
    - webpack
---

[![](http://ou8vcvyuy.bkt.clouddn.com/dive-into-webpack-for-blog.jpg)](http://webpack.wuhaolin.cn/)


webpack是一个js打包工具，不一个完整的前端构建工具。它的流行得益于模块化和单页应用的流行。webpack提供扩展机制，在庞大的社区支持下各种场景基本它都可找到解决方案。本文的目的是教会你用webpack解决实战中常见的问题。

## webpack原理
在深入实战前先要知道webpack的运行原理
### webpack核心概念
- `entry` 一个可执行模块或库的入口文件。
- `chunk` 多个文件组成的一个代码块，例如把一个可执行模块和它所有依赖的模块组合和一个 `chunk` 这体现了webpack的打包机制。
- `loader` 文件转换器，例如把es6转换为es5，scss转换为css。
- `plugin` 插件，用于扩展webpack的功能，在webpack构建生命周期的节点上加入扩展hook为webpack加入功能。

### webpack构建流程
从启动webpack构建到输出结果经历了一系列过程，它们是：
1. 解析webpack配置参数，合并从shell传入和`webpack.config.js`文件里配置的参数，生产最后的配置结果。
2. 注册所有配置的插件，好让插件监听webpack构建生命周期的事件节点，以做出对应的反应。
3. 从配置的`entry`入口文件开始解析文件构建AST语法树，找出每个文件所依赖的文件，递归下去。
4. 在解析文件递归的过程中根据文件类型和loader配置找出合适的loader用来对文件进行转换。
5. 递归完后得到每个文件的最终结果，根据`entry`配置生成代码块`chunk`。
6. 输出所有`chunk`到文件系统。

需要注意的是，在构建生命周期中有一系列插件在合适的时机做了合适的事情，比如`UglifyJsPlugin`会在loader转换递归完后对结果再使用`UglifyJs`压缩覆盖之前的结果。

## 场景和方案
通过各种场景和对应的解决方案让你深入掌握webpack
### 单页应用
**demo [redemo](https://github.com/gwuhaolin/redemo)**
一个单页应用需要配置一个`entry`指明执行入口，webpack会为`entry`生成一个包含这个入口所有依赖文件的`chunk`，但要让它在浏览器里跑起来还需要一个HTML文件来加载`chunk`生成的js文件，如果提取出了css还需要让HTML文件引入提取出的css。[web-webpack-plugin](https://github.com/gwuhaolin/web-webpack-plugin)里的`WebPlugin`可以自动的完成这些工作。

webpack配置文件
```js
const { WebPlugin } = require('web-webpack-plugin');
module.exports = {
  entry: {
    app: './src/doc/index.js',
  },
  plugins: [
    // 一个WebPlugin对应生成一个html文件
    new WebPlugin({
      //输出的html文件名称
      filename: 'index.html',
      //这个html依赖的`entry`
      requires: ['app'],
    }),
  ],
};
```
`requires: ['doc']`指明这个HTML依赖哪些`entry`，`entry`生成的js和css会自动注入到HTML里。
你还可以配置这些资源的注入方式，支持如下属性：
- `_dist` 只有在生产环境下才引入该资源
- `_dev` 只有在开发环境下才引入该资源
- `_inline` 把该资源的内容潜入到html里
- `_ie` 只有IE浏览器才需要引入的资源

要设置这些属性可以通过在js里配置
```js
new WebPlugin({
    filename: 'index.html',
    requires: {
         app:{
              _dist:true,
              _inline:false,
         }
    },
}),
```
或者在模版里设置，使用模版的好处是灵活的控制资源注入点。
```js
new WebPlugin({
      filename: 'index.html',
      template: './template.html',
}),
```
```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
    <link rel="stylesheet" href="app?_inline">
    <script src="ie-polyfill?_ie"></script>
</head>
<body>
<div id="react-body"></div>
<script src="app"></script>
</body>
</html>
```
`WebPlugin`插件借鉴了`fis3`的思想，补足了webpack缺失的以HTML为入口的功能。想了解`WebPlugin`的更多功能，见[文档](https://github.com/gwuhaolin/web-webpack-plugin/blob/master/readme_zh.md#输出html文件-demo)。

### 一个项目里管理多个单页应用
一般项目里会包含多个单页应用，虽然多个单页应用也可以合并成一个但是这样做会导致用户没访问的部分也加载了。如果项目里有很多个单页应用，为每个单页应用配置一个`entry`和`WebPlugin `？如果项目又新增了一个单页应用，又去新增webpack配置？这样做太麻烦了，[web-webpack-plugin](https://github.com/gwuhaolin/web-webpack-plugin)里的`AutoWebPlugin`可以方便的解决这些问题。
```js
module.exports = {
    plugins: [
        // 所有页面的入口目录
        new AutoWebPlugin('./src/'),
    ]
};
```
`AutoWebPlugin`会把`./src/`目录下所有每个文件夹作为一个单页页面的入口，自动为所有的页面入口配置一个WebPlugin输出对应的html。要新增一个页面就在`./src/`下新建一个文件夹包含这个单页应用所依赖的代码，`AutoWebPlugin`自动生成一个名叫文件夹名称的html文件。`AutoWebPlugin`的更多功能见[文档](https://github.com/gwuhaolin/web-webpack-plugin/blob/master/readme_zh.md#自动探测html入口-demo)。

### 代码分割优化
一个好的代码分割对浏览器首屏效果提升很大。比如对于最常见的react体系你可以
1. 先抽出基础库`react` `react-dom` `redux` `react-redux`到一个单独的文件而不是和其它文件放在一起打包为一个文件，这样做的好处是只要你不升级他们的版本这个文件永远不会被刷新。如果你把这些基础库和业务代码打包在一个文件里每次改动业务代码都会导致文件hash值变化从而导致缓存失效浏览器重复下载这些包含基础库的代码。以上的配置为：
```js
// vender.js 文件抽离基础库到单独的一个文件里防止跟随业务代码被刷新
// 所有页面都依赖的第三方库
// react基础
import 'react';
import 'react-dom';
import 'react-redux';
// redux基础
import 'redux';
import 'redux-thunk';
```
```js
// webpack配置
{
  entry: {
    vendor: './path/to/vendor.js',
  },
}
```
2. 再通过[CommonsChunkPlugin](https://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin)可以提取出多个代码块都依赖的代码形成一个单独的`chunk`。在应用有多个页面的场景下提取出所有页面公共的代码减少单个页面的代码，在不同页面之间切换时所有页面公共的代码之前被加载过而不必重新加载。

### 构建npm包
**demo [remd](https://github.com/gwuhaolin/remd)**
除了构建可运行的web应用，webpack也可用来构建发布到npm上去的给别人调用的js库。
```js
const nodeExternals = require('webpack-node-externals');
module.exports = {
  entry: {
    index: './src/index.js',
  },
  externals: [nodeExternals()],
  target: 'node',
  output: {
    path: path.resolve(__dirname, '.npm'),
    filename: '[name].js',
    libraryTarget: 'commonjs2',
  },
};
```
这里有几个区别于web应用不同的地方：
- `externals: [nodeExternals()]`用于排除`node_modules`目录下的代码被打包进去，因为放在`node_modules`目录下的代码应该通过npm安装。
- `libraryTarget: 'commonjs2'`指出`entry`是一个可供别人调用的库而不是可执行的，输出的js文件按照commonjs规范。

### 构建服务端渲染
服务端渲染的代码要运行在nodejs环境，和浏览器不同的是，服务端渲染代码需要采用commonjs规范同时不应该包含除js之外的文件比如css。webpack配置如下：
```js
module.exports = {
  target: 'node',
  entry: {
    'server_render': './src/server_render',
  },
  output: {
    filename: './dist/server/[name].js',
    libraryTarget: 'commonjs2',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
      },
      {
        test: /\.(scss|css|pdf)$/,
        loader: 'ignore-loader',
      },
    ]
  },
};
```
其中几个关键的地方在于：
- `target: 'node'` 指明构建出的代码是要运行在node环境里
- `libraryTarget: 'commonjs2'` 指明输出的代码要是commonjs规范
- `{test: /\.(scss|css|pdf)$/,loader: 'ignore-loader'}` 是为了防止不能在node里执行服务端渲染也用不上的文件被打包进去。

### 从fis3迁移到webpack
fis3和webpack有相似的地方也有不同的地方。相似在于他们都采用commonjs规范，不同在于导入css这些非js资源的方式。fis3通过`// @require './index.scss'`而webpack通过`require('./index.scss')`。如果想从fis3平滑迁移到webpack可以使用[comment-require-loader](https://github.com/gwuhaolin/comment-require-loader/issues)。比如你想在webpack构建是使用采用了fis3方式的`imui`模块，配置如下：
```js
loaders:[{
     test: /\.js$/,
     loaders: ['comment-require-loader'],
     include: [path.resolve(__dirname, 'node_modules/imui'),]
}]
```

## 自定义webpack扩展
如果你在社区找不到你的应用场景的解决方案，那就需要自己动手了写loader或者plugin了。
在你编写自定义webpack扩展前你需要想明白到底是要做一个`loader`还是`plugin`呢？可以这样判断：
> 如果你的扩展是想对一个个单独的文件进行转换那么就编写`loader`剩下的都是`plugin`。

其中对文件进行转换可以是像：
- `babel-loader`把es6转换成`es5`
- `file-loader`把文件替换成对应的URL
- `raw-loader`注入文本文件内容到代码里去

### 编写 webpack loader
**demo [comment-require-loader](https://github.com/gwuhaolin/comment-require-loader)**
编写`loader`非常简单，以comment-require-loader为例：
```js
module.exports = function (content) {
    return replace(content);
};
```
`loader`的入口需要导出一个函数，这个函数要干的事情就是转换一个文件的内容。
函数接收的参数`content`是一个文件在转换前的字符串形式内容，需要返回一个新的字符串形式内容作为转换后的结果，所有通过模块化倒入的文件都会经过`loader`。从这里可以看出`loader`只能处理一个个单独的文件而不能处理代码块。想编写更复杂的loader可参考[官方文档](https://webpack.github.io/docs/loaders.html)

### 编写 webpack plugin
**demo [end-webpack-plugin](https://github.com/gwuhaolin/end-webpack-plugin)**
`plugin`应用场景广泛，所以稍微复杂点。以end-webpack-plugin为例：
```js
class EndWebpackPlugin {

    constructor(doneCallback, failCallback) {
        this.doneCallback = doneCallback;
        this.failCallback = failCallback;
    }

    apply(compiler) {
        // 监听webpack生命周期里的事件，做相应的处理
        compiler.plugin('done', (stats) => {
            this.doneCallback(stats);
        });
        compiler.plugin('failed', (err) => {
            this.failCallback(err);
        });
    }
}

module.exports = EndWebpackPlugin;
```
`loader`的入口需要导出一个class, 在`new EndWebpackPlugin()`的时候通过构造函数传入这个插件需要的参数，在webpack启动的时候会先实例化`plugin`再调用`plugin`的`apply`方法，插件需要在`apply`函数里监听webpack生命周期里的事件，做相应的处理。
webpack plugin 里有2个核心概念：
- `Compiler`: 从webpack启动到推出只存在一个`Compiler`，`Compiler`存放着webpack配置
- `Compilation`: 由于webpack的监听文件变化自动编译机制，`Compilation`代表一次编译。

`Compiler` 和 `Compilation` 都会广播一系列事件。
webpack生命周期里有非常多的事件可以在[event-hooks](https://webpack.js.org/api/plugins/compiler/#event-hooks)和[Compilation](https://webpack.js.org/api/plugins/compilation/)里查到。以上只是一个最简单的demo，更复杂的可以查看 [how to write a plugin](https://github.com/webpack/docs/wiki/how-to-write-a-plugin)或参考[web-webpack-plugin](https://github.com/gwuhaolin/web-webpack-plugin)。

## 总结
webpack其实很简单，可以用一句话涵盖它的本质：
> webpack是一个打包模块化js的工具，可以通过loader转换文件，通过plugin扩展功能。

如果webpack让你感到复杂，一定是各种loader和plugin的原因。
希望本文能让你明白webpack的原理与本质让你可以在实战中灵活应用webpack。

[阅读原文](http://wuhaolin.cn/2017/05/31/webpack%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%98/)
