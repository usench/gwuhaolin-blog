---
title: ReactNative三端同构实战
date: 2018-11-14T04:21:47Z
url: https://github.com/gwuhaolin/blog/issues/18
tags:
    - react
---

## 认识ReactNative三端同构

ReactNative三端同构是指在不改动原ReactNative的代码下，让其在浏览器中运行出和在ReactNative环境下一样的页面。

ReactNative三端同构的应用场景包括：

- 在ReactNative页面崩溃时用对应的Web页兜底，以保证用户可以正常使用页面；
- 对于需要分享到社交网络的页面，例如需要分享到微信朋友圈、微博的页面，不可避免的需要Web网页。

对于使用ReactNative开发的页面，如果又单独为Web平台重复写一份代码代价是极其大的，而ReactNative三端同构能以零花费快速做到一份代码三端复用。

## ReactNative三端同构基础原理

ReactNative就像一套新的浏览器标准，ReactNative提供了大量内置的原生UI元素和系统API，对应着浏览器中的div、img等标签以及BOM API；但是ReactNative目前只专注于移动App平台，只适配了iOS和Android两大系统，而浏览器则是适配了各种操作系统，由于ReactNative需要适配的平台更少所以性能会比浏览器要好。

我们编写的React组件经过render后会以虚拟DOM的形式存储在内存中，React只负责UI层面的抽象和组件的状态管理，各平台都可用虚拟DOM去渲染出不同的结果，React架构如下：

<img width="719" alt="react" src="https://user-images.githubusercontent.com/5773264/48459964-77f8a880-e807-11e8-9e5f-37f4145e7efe.png">


由此可见虚拟DOM这层中间抽象在实现React渲染到多端时发挥了很大的作用。

## ReactNative三端同构方案对比

目前社区中已经有多个ReactNative三端同构方案，比较成熟的有[react-native-web](https://github.com/necolas/react-native-web)和[reactxp](https://github.com/Microsoft/reactxp)，下来从多方面对比二者以帮助你做出合适的选择。

#### 认识reactxp

reactxp是一个跨平台的UI库，由微软Skype团队维护和开源，Skype产品中就大量使用了它来实现写一份代码运行在多个平台上。目前reactxp支持以下平台：

- iOS（基于react-native渲染）；
- Android（基于react-native渲染）；
- Web（基于react-dom渲染）；
- [UWP](https://en.wikipedia.org/wiki/Universal_Windows_Platform) (基于[react-native-windows](https://github.com/Microsoft/react-native-windows)渲染)；
- 对于剩下的平台，诸如Mac、Windows10以下系统、Linux桌面，则采用基于Web渲染的[Electron](https://electron.atom.io/)。

#### reactxp实现原理

reactxp充份发挥了react虚拟DOM的优势，它其实只是充当胶水的作用，把各个平台的渲染引擎整合起来，对外暴露平台一致的接口。

reactxp为各个平台都实现了一份代码，在构建的过程中构建工具会自动选择平台相关的代码进行打包输出。

![reactxp arch](https://user-images.githubusercontent.com/5773264/48459985-88a91e80-e807-11e8-81ee-8d5e0f3856eb.png)


#### reactxp和react-native的异同点

从使用层面来说它们最大的区别在于：**reactxp是为了一份代码在多个平台运行，而react-native是为了学一遍可为多个平台编写原生应用**。

这一点从reactxp和react-native暴露出的API就可以看出来：react-native中大量诸如SegmentedControlIOS、PermissionsAndroid这样针对特定平台的API，而reactxp中所有的API在所有端中都可以正常调用。

事实上react-native也在为多端接口统一做努力，react-native中的大多数接口是可以在多端运行一致的，但为了保证灵活性react-native也提供了平台相关的接口。而reactxp磨平了多端接口的差异，但这也导致reactxp灵活性降低。

他们的相同点是都采用了react框架编程的思想，由于reactxp是基于react-native封装的导致他们大多数API的使用方式都是一致的。

#### react-native-web原理

react-native-web实现了在不修改react-native代码的情况下渲染在浏览器里的功能，其实现原理如下：

在用webpack构建用于运行在浏览器里的代码时，会把react-native的导入路径替换为react-native-web的导入路径，在react-native-web内部则会以和react-native目录结构一致的方式实现了一致的react-native组件。在react-native-web组件的内部，则把react-native的API映射成了浏览器支持的API。

![react-native-web arch](https://user-images.githubusercontent.com/5773264/48459994-952d7700-e807-11e8-976f-4e4777a8b1c5.png)


#### react-native-web和reactxp异同点

它们的目的都是为了实现多端同构，但react-native-web只专注于Web平台的适配，而reactxp则还需要适配UWP平台。

在实现Web平台的适配过程中它们都采用了类似的原理：把对外暴露的API或组件映射到Web平台去。

但在实现Web平台的样式适配时有细微区别：

- reactxp全部通过内联样式实现；
- react-native-web通过为每条不同的样式生产一个className，对于重复出现的样式则通过复用className实现。

对于这两种不同的实现方式，我更看好react-native-web的实现方式，原因有两个：

1. 通过复用className节省网络传输字节，如果你需要做服务端渲染这个优势会凸显出来；
2. 通过className的方式浏览器渲染性能更好，原因是浏览器有做样式计算缓存优化，有人专门写了[性能对比测试页面](https://jsperf.com/inline-style-vs-css-class/2)。

#### reactxp优点

- 写一份代码就可实现多端渲染，对于有多端需求的产品可以减少代码量和人力；
- 由微软Skype团队维护并且用于Skype产品中，有大公司投入资源支持；
- 基于TypeScript编写，对IDE友好；

#### reactxp缺点

- 为了抹平多端平台差异导致灵活性降低，暴露的组件和API较react-native要少很多；
- 需要兼容UWP平台导致包袱更重，而目前会针对Windows桌面或手机开发应用的产品也再渐渐减少，大多数产品不需要支持Windows平台；
- 需要多导入reactxp这个库，导致打包输出的bundle会变大；并且由于多了一层适配，运行时性能肯定是不如直接使用react-native。

其中最为致命的缺点可能在于目前reactxp支持的组件和API相当匮乏，一些比较细的操作无法控制；如果你在reactxp项目中确实有需求超出reactxp的能力范围，可以通过导入和使用react-native实现，但这会导致整个项目脱离reactxp体系，因此reactxp为我们实现的多端同构将会无法实现；

reactxp只保证在它的体型内实现多端同构，但在它的体系内却有很多API不可用。

reactxp更像是微软为了挽救其奄奄一息的[Windows Phone](https://en.wikipedia.org/wiki/Windows_Phone)系统在做努力，但事实上微软已经失去了移动操作系统市场，无人愿意为用户量很少的WP系统开发APP。

#### react-native-web和reactxp对比表

|                | reactxp                                                      | react-native-web                                             | 对比                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 维护人         | 微软Skype团队和GitHub社区                                    | 来自Twitter的个人[necolas](https://github.com/necolas)和GitHub社区 | reactxp小胜                                                  |
| 服务端渲染支持 | [官方没有明确要支持](https://github.com/Microsoft/reactxp/issues/201) | 完全支持                                                     | react-native-web胜                                           |
| Web端包大小    | 435KB                                                        | 354.4KB                                                      | react-native-web胜                                           |
| 写代码效率     | 针对reactxp暴露的API去实现多端适配                           | 需要自己去验证代码在多端的表现是否一致                       | reactxp胜                                                    |
| 学习成本       | 除了需要学习reactxp外，不可避免的还需要学习react-native      | 只需学习react-native即可                                     | react-native-web胜                                           |
| Github数据     | start=2017年4月 star=6521 issues=23/739 commits=814          | start=2017年7月 star=10151 issues=45/1034 commits=1248       | react-native-web用户更多，代码变动频率更大。reactxp问题响应速度更快。 |

#### 如何选择

如果你开发的产品有适配UWP平台的需求就选择reactxp，否则选择react-native-web，因为reactxp相比于react-native-web除了多支持Windows平台外，并无其它明显优势。

## react-native-web接入

为了给你现有的ReactNative接入react-native-web，实现ReactNative三端同构的能力，你需要做以下事情：

1. 安装新的依赖：

   ```base
   # 运行时依赖
   npm i react react-dom react-native-web react-art
   # 构建工具
   npm i -D webpack webpack-dev-server webpack-cli babel-loader babel-plugin-transform-runtime
   ```

2. 为Web平台写一份Webpack配置文件webpack.config.js，内容如下:

   ```js
   module.exports = {
     module: {
       rules: [
         {
           // 支持图片等静态文件的加载
           test: /\.(gif|jpe?g|png|svg)$/,
           use: {
             loader: 'file-loader'
           }
         },
         {
   		// react-native包中有很多es6语法的js，需要用babel转换后才能在浏览器中运行
           test: /\.js$/,
           use: {
             loader: 'babel-loader',
             options: {
               cacheDirectory: false,
               presets: ['react-native'],
               plugins: [
                 // 支持 async/await 语法
                 'transform-runtime'
               ]
             }
           }
         }
       ]
     },
     resolve: {
       // 优先加载以web.js结尾的针对web平台的文件
       extensions: {
           '.web.js',
           '.js',
           '.json',
       },
       alias: {
          // 把react-native包映射成react-native-web
         'react-native$': 'react-native-web'
       }
     }
   }
   ```

3. 写一个针对Web平台启动入口文件index.web.js：

   ```js
   import { AppRegistry } from 'react-native';
   
   // 注册组件
   AppRegistry.registerComponent('App', () => App);
   
   // 启动App组件
   AppRegistry.runApplication('App', {
     // 启动时传给App组件的属性
     initialProps: {},
     // 渲染App的DOM容器
     rootTag: document.getElementById('react-app')
   });
   ```

4. 写一个index.html文件，引入Webpack构建出的JavaScript，以在Web平台运行：

   ```html
   <html>
   <head>
       <meta charset="UTF-8">
       <meta name="viewport"
             content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
       <meta http-equiv="X-UA-Compatible" content="ie=edge">
       <!--以下是正常运行所需的必须样式-->
       <style>
           html,body,#react-root{
               height: 100%;
           }
           #react-root{
               display:flex;
           }
       </style>
   </head>
   <body>
   <div id="react-root"></div>
   <script src="main.js"></script>
   </body>
   </html>
   ```

完成以上步骤后重新执行webpack构建，再在浏览器中打开index.html你就可以看到ReactNative转出的Web网页了。

完整的例子可以参考react-native-web的[官方例子](https://github.com/necolas/react-native-web/tree/master/packages/examples)。

## reactxp接入

由于reactxp所有暴露的API都是支持在Web平台和ReactNative平台同时正常运行的，因此为reactxp应用转Web的方法非常简单，只需为项目加入Webpack构建和运行Web页面的index.html文件。

Webpack配置文件如下：

```js
module.exports = {
  entry: "./src/index.tsx",
  mode: "development",
  output: {
    filename: "bundle.js",
    path: __dirname + "/dist"
  },
  resolve: {
    // 优先加载web.js后缀的文件
    extensions: [".web.js", ".ts", ".tsx", ".js"]
  },

  module: {
    rules: [
  	  // 转换TypeScript文件    
      { test: /\.tsx?$/, loader: "awesome-typescript-loader" }
    ]
  }
};
```

再写一个运行Web页面的index.html文件：

```html
<!doctype html>
<html>
<head>
  <meta charset='utf-8'>
  <style>
    html, body, .app-container {
      width: 100%;
      height: 100%;
      padding: 0;
      border: none;
      margin: 0;
    }
    *:focus {
        outline: 0;
    }
  </style>
</head>
<body>
  <div class="app-container"></div>
  <script src="dist/bundle.js"></script>
</body>
</html>
```

完整的例子可以参考reactxp的[官方例子](https://github.com/Microsoft/reactxp/tree/master/samples/hello-world)。

## 适配你项目中自定义的NativeModules

ReactNative开发的App中经常会出现ReactNative官方提供的NativeModules不够用的情况，这时你会在项目中开发自己的NativeModules，然后在JavaScript中去调用自己的NativeModules。这在ReactNative环境下运行没有问题，但转成Web后执行时会报错说NativeModules上找不到对应的模块，这时因为在浏览器环境下是不存在这些自定义的NativeModules。为了让页面能正常在浏览器中运行，需要为Web平台也实现一份自定义的NativeModules，实现方法可以web平台的执行入口的最开头注入以下polyfill：

```js
import { NativeModules } from 'react-native';
import MyModule from './MyModule'; // 实现自定义NativeModules的地方

NativeModules.MyModule = MyModule; // 挂载MyModule
```

这段代码的作用是把针对Web平台编写的自定义原生模块挂载到NativeModules对象上成为其属性，以让JavaScript代码在访问自定义NativeModules时访问到针对Web平台编写模块。

## 编写特定平台的代码

为了让ReactNative三端同构能正常的运行，在有些情况下你不得不编写平台特点的代码，因为有些代码只能在特点平台下才能运行，编写Web平台特定的代码的方法有以下三种：

1. ReactNative.Platform.OS：所有端的代码都在一个文件中，通过以下代码来写web平台专属代码:

   ```js
   import { Platform } from 'react-native';
   
   if(Platform.OS==='web'){
     // web平台专属代码
   }
   ```

2. process.env.platform：通过Webpack注入的环境变量来区分

   ```js
   if (process.env.platform === 'web') {
     // web平台专属代码
   }
   ```

   这段代码只会在web平台下被打包进去，这和`ReactNative.Platform`的区别是：

   `ReactNative.Platform`的代码会打包进所有的平台。

   要使用这种方法需要你在webpack.config.js文件中注入环境变量：

   ```js
   plugins: [
   	new webpack.DefinePlugin({
   		'process.env': {
   			platform: JSON.stringify(platform),
   			__DEV__: mode === 'development'
   	}),
   ]
   ```

3. .web.js: 在web模式下会优先加载.web.js文件，当.web.js文件不存在时才使用.js文件。

## 总结

ReactNative三端同构在理论上虽然可行，并且有现成的方案，但实践是还是会遇到一些问题，例如：

- 在Web平台运行出的样式和ReactNative平台不一致，针对这种情况一般是react-native-web库的适配问题，可以在github上提issus或pr；
- 有些ReactNative提供的API在Web平台不可能实现适配，例如调摄像头、振动等，对于这种问题只有在Web平台裁剪掉这些功能或使用其它交互方式替代。

ReactNative三端同构虽然无法实现100%和ReactNative环境运行一致，但能快速简单的转换大多数场景，以低成本的方式为你的项目带来收益。


> 本文首发于[IBM Dev社区](https://www.ibm.com/developerworks/cn/web/wa-universal-react-native/index.html)


