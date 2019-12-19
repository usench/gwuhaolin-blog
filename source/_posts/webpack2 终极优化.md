---
title: webpack2 终极优化
date: 2017-04-30T09:51:14Z
url: https://github.com/gwuhaolin/blog/issues/2
tags:
    - webpack
---

[![](http://ou8vcvyuy.bkt.clouddn.com/dive-into-webpack-for-blog.jpg)](http://webpack.wuhaolin.cn/)

webpack是当下最流行的js打包工具，这得益于网页应用日益复杂和js模块化的流行。[webpack2](https://webpack.js.org)增加了一些新特性也正式发布了一段时间，是时候告诉大家如何用webpack2优化你的构建让它构建出更小的文件尺寸和更好的开发体验。

# 优化输出
打包结果更小可以让网页打开速度更快以及简约宽带。可以通过这以下几点做到

#### 压缩css
`css-loader` 在webpack2里默认是没有开启压缩的，最后生成的css文件里有很多空格和tab，通过配置
`css-loader?minimize`参数可以开启压缩输出最小的css。css的压缩实际是是通过[cssnano](http://cssnano.co)实现的。

#### tree-shaking
tree-shaking 是指借助es6 `import export` 语法静态性的特点来删掉export但是没有import过的东西。要让tree-shaking工作需要注意以下几点：
- 配置babel让它在编译转化es6代码时不把`import export`转换为cmd的`module.export`，配置如下：
```json
"presets": [
    [
      "es2015",
      {
        "modules": false
      }
    ]
]
```
- 大多数分布到npm的库里的代码都是es5的，但是也有部分库（redux,react-router等等）开始支持tree-shaking。这些库发布到npm里的代码即包含es5的又包含全采用了es6 `import export` 语法的代码。
拿redux库来说，npm下载到的目录结构如下：
```
├── es
│   └── utils
├── lib
│   └── utils
```
其中lib目录里是编译出的es5代码，es目录里是编译出的采用`import export` 语法的es5代码，在redux的`package.json`文件里有这两个配置：
```
"main": "lib/index.js",
"jsnext:main": "es/index.js",
```
这是指这个库的入口文件的位置，所以要让webpack去读取es目录下的代码需要使用jsnext:main字段配置的入口，要做到这点webpack需要这样配置：
```js
module.exports = {
	resolve: {
            mainFields: ['jsnext:main','main'],
        }
};
```
这会让webpack先使用jsnext:main字段，在没有时使用main字段。这样就可以优化支持tree-shaking的库。

#### 优化 UglifyJsPlugin
webpack `--optimize-minimize` 选项会开启 UglifyJsPlugin来压缩输出的js，但是默认的UglifyJsPlugin配置并没有把代码压缩到最小输出的js里还是有注释和空格，需要覆盖默认的配置：
```js
new UglifyJsPlugin({
    // 最紧凑的输出
    beautify: false,
    // 删除所有的注释
    comments: false,
    compress: {
      // 在UglifyJs删除没有用到的代码时不输出警告  
      warnings: false,
      // 删除所有的 `console` 语句
      // 还可以兼容ie浏览器
      drop_console: true,
      // 内嵌定义了但是只用到一次的变量
      collapse_vars: true,
      // 提取出出现多次但是没有定义成变量去引用的静态值
      reduce_vars: true,
    }
})
```

#### 定义环境变量 NODE_ENV=production
很多库里（比如react）有部分代码是这样的：
```js
if(process.env.NODE_ENV !== 'production'){
// 不是生产环境才需要用到的代码，比如控制台里看到的警告    
}
```
在环境变量 `NODE_ENV` 等于 `production` 的时候UglifyJs会认为if语句里的是死代码在压缩代码时删掉。

#### 使用 CommonsChunkPlugin 抽取公共代码
[CommonsChunkPlugin](https://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin)可以提取出多个代码块都依赖的模块形成一个单独的模块。要发挥CommonsChunkPlugin的作用还需要浏览器缓存机制的配合。在应用有多个页面的场景下提取出所有页面公共的代码减少单个页面的代码，在不同页面之间切换时所有页面公共的代码之前被加载过而不必重新加载。这个方法可以非常有效的提升应用性能。

#### 在生产环境按照文件内容md5打hash
webpack编译在生产环境出来的js、css、图片、字体这些文件应该放到CDN上，再根据文件内容的md5命名文件，利用缓存机制用户只需要加载一次，第二次加载时就直接访问缓存。如果你之后有修改就会为对应的文件生产新的md5值。做到以上你需要这样配置：
```js
{
  output: {
    publicPath: CND_URL,
    filename: '[name]_[chunkhash].js',
  },
}
```
知道以上原理后我们还可以进一步优化：利用CommonsChunkPlugin提取出使用页面都依赖的基础运行环境。比如对于最常见的react体系你可以抽出基础库`react` `react-dom` `redux` `react-redux`到一个单独的文件而不是和其它文件放在一起打包为一个文件，这样做的好处是只要你不升级他们的版本这个文件永远不会被刷新。如果你把这些基础库和业务代码打包在一个文件里每次改动业务代码都会导致浏览器重复下载这些包含基础库的代码。以上的配置为：
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

#### DedupePlugin 和 OccurrenceOrderPlugin
在webpack1里经常会使用 `DedupePlugin` 插件来消除重复的模块以及使用 `OccurrenceOrderPlugin` 插件让被依赖次数更高的模块靠前分到更小的id 来达到输出更少的代码，在webpack2里这些已经这两个插件已经被移除了因为这些功能已经被内置了。

除了压缩文本代码外还可以：
- **用[imagemin-webpack-plugin](https://github.com/Klathmon/imagemin-webpack-plugin) 压缩图片**
- **用[webpack-spritesmith](https://github.com/mixtur/webpack-spritesmith) 合并雪碧图**
- **对于支持es6的js运行环境使用[babili](https://github.com/babel/babili)**

以上优化点只需要在构建用于生产环境代码的时候才使用，在开发环境时最好关闭因为它们很耗时。



# 优化开发体验
优化开发体验主要从更快的构建和更方便的功能入手。

##  更快的构建

#### 缩小文件搜索范围
webpack的`resolve.modules`配置模块库（通常是指node_modules）所在的位置，在js里出现`import 'redux'`这样不是相对也不是绝对路径的写法时会去node_modules目录下找。但是默认的配置会采用向上递归搜索的方式去寻找node_modules，但通常项目目录里只有一个node_modules在项目根目录，为了减少搜索我们直接写明node_modules的全路径：
```js
module.exports = {
    resolve: {
        modules: [path.resolve(__dirname, 'node_modules')]
    }
};
```

除此之外webpack配置loader时也可以缩小文件搜索范围。
- loader的test正则表达式也应该尽可能的简单，比如在你的项目里只有`.js`文件时就不要把test写成`/\.jsx?$/`
- loader使用include命中只需要处理的文件，比如babel-loader的这两个配置:

只对项目目录下src目录里的代码进行babel编译
```js
{
    test: /\.js$/,
    loader: 'babel-loader',
    include: path.resolve(__dirname, 'src')
}	
```
项目目录下的所有js都会进行babel编译，包括庞大的node_modules下的js
```js
{
    test: /\.js$/,
    loader: 'babel-loader'
}	
```

#### 开启 babel-loader 缓存
babel编译过程很耗时，好在babel-loader提供缓存编译结果选项，在重启webpack时不需要创新编译而是复用缓存结果减少编译流程。babel-loader缓存机制默认是关闭的，打开的配置如下：
```js
module.exports = {
    module: {
         loaders: [{
                test: /\.js$/,
                loader: 'babel-loader?cacheDirectory',
         }]
  }
};
```

#### 使用 alias
`resolve.alias` 配置路径映射。
发布到npm的库大多数都包含两个目录，一个是放着cmd模块化的lib目录，一个是把所有文件合成一个文件的dist目录，多数的入口文件是指向lib里面下的。
默认情况下webpack会去读lib目录下的入口文件再去递归加载其它依赖的文件这个过程很耗时，alias配置可以让webpack直接使用dist目录的整体文件减少文件递归解析。配置如下：
```js
module.exports = {
  resolve: {
    alias: {
      'moment': 'moment/min/moment.min.js',
      'react': 'react/dist/react.js',
      'react-dom': 'react-dom/dist/react-dom.js'
    }
  }
};
```

#### 使用 noParse
`module.noParse` 配置哪些文件可以脱离webpack的解析。
有些库是自成一体不依赖其他库的没有使用模块化的，比如jquey、momentjs、chart.js，要使用它们必须整体全部引入。
webpack是模块化打包工具完全没有必要去解析这些文件的依赖，因为它们都不依赖其它文件体积也很庞大，要忽略它们配置如下：
```js
module.exports = {
  module: {
    noParse: /node_modules\/(jquey|moment|chart\.js)/
  }
};
```

除此以外还有很多可以加速的方法：
- **使用[happypack](https://github.com/amireh/happypack)多进程并行构建**
- **使用[DllPlugin](https://github.com/webpack/docs/wiki/list-of-plugins#dllplugin)复用模块**

## 更方便的功能 

#### 模块热替换
模块热替换是指在开发的过程中修改代码后不用刷新页面直接把变化的模块替换到老模块让页面呈现出最新的效果。
webpack-dev-server内置模块热替换，配置起来也很方便，下面以react应用为例，步骤如下：
- 在启动webpack-dev-server的时候带上`--hot`参数开启模块热替换，在开启`--hot`后针对css的变化是会自动热替换的，但是js涉及到复杂的逻辑还需要进一步配置。
- 配置页面入口文件

```js
import App from './app';

function run(){
	render(<App/>,document.getElementById('app'));
}
run();

// 只在开发模式下配置模块热替换
if (process.env.NODE_ENV !== 'production') {
  module.hot.accept('./app', run);
}
```

当./app发生变化或者当./app依赖的文件发生变化时会把./app编译成一个模块去替换老的，替换完毕后重新执行run函数渲染出最新的效果。

#### 自动生成html
webpack只做了资源打包的工作还缺少把这些加载到html里运行的功能，在庞大的app里手写html去加载这些资源是很繁琐易错的，我们需要自动正确的加载打包出的资源。
webpack原生不支持这个功能于是我做了一个插件 [web-webpack-plugin](https://github.com/gwuhaolin/web-webpack-plugin)
具体使用点开链接看[详细文档](https://github.com/gwuhaolin/web-webpack-plugin/blob/master/readme_zh.md)，使用大概如下：

[demo](https://github.com/gwuhaolin/web-webpack-plugin/tree/master/demo/out-html)

webpack配置
```js
module.exports = {
    entry: {
        A: './a',
        B: './b',
    },
    plugins: [
        new WebPlugin({
            // 输出的html文件名称，必填，注意不要重名，重名会覆盖相互文件。
            filename: 'index.html',
            // 该html文件依赖的entry，必须是一个数组。依赖的资源的注入顺序按照数组的顺序。
            requires: ['A', 'B'],
        }),
    ]
};
```

将会输出一个`index.html`文件，这个文件将会自动引入 entry `A` 和 `B` 生成的js文件，

输出的html:
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
</head>
<body>
<script src="A.js"></script>
<script src="B.js"></script>
</body>
</html>
```

输出的目录结构
```
├── A.js
├── B.js
└── index.html
```

#### 管理多页面
虽然webpack适用于单页应用，但复杂的系统经常是由多个单页应用组成，每个页面一个功能模块。webpack给出了js打包方案但缺少管理多个页面的功能。 [web-webpack-plugin](https://github.com/gwuhaolin/web-webpack-plugin)的`AutoWebPlugin`会自动的为你的系统里每个单页应用生成一个html入口页，这个入口会自动的注入当前单页应用依赖的资源，使用它你只需如下几行代码：
```js
plugins: [
    // ./src/pages/ 代表存放所有页面的根目录，这个目录下的每一个目录被看着是一个单页应用
    // 会为里面的每一个目录生成一个html入口
    new AutoWebPlugin('./src/pages/', {
      //使用单页应用的html模版文件，这里你可以自定义配置
      template: './src/assets/template.html',
    }),
],
```
查看web-webpack-plugin的[文档了解更多](https://github.com/gwuhaolin/web-webpack-plugin/blob/master/readme_zh.md#自动探测html入口-demo)

### 分析输出结果
如果你对当前的配置输出或者构建速度不满意，webpack有一个工具叫做[webpack analyze](https://webpack.github.io/analyse/) 以可视化的方式直观的分析构建，来进一步优化构建结果和速度。要使用它你需要在执行webpack的时候带上`--json --profile`2个参数，这代表让webpack把构建结果以json输出并带上构建性能信息，使用如下：
```bash
webpack --json --profile > stats.json 
```
会生产一个`stats.json `文件，再打开[webpack analyze](https://webpack.github.io/analyse/) 上传这个文件开始分析。

**最后附上这篇文章所讲到的[webpack整体的配置](https://gist.github.com/gwuhaolin/cebd252a23793e742e6acae90ab63e83)，分为开发环境的`webpack.config.js`和生产环境的`webpack-dist.config.js`**

[阅读原文](http://wuhaolin.cn/2017/04/30/webpack2%20%E7%BB%88%E6%9E%81%E4%BC%98%E5%8C%96/)
