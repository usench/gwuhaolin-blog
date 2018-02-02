---
title: 快速优雅的为React组件生成文档
date: 2017-04-30T09:41:44Z
url: https://github.com/gwuhaolin/blog/issues/1
tags:
    - react
---

在开发React组件时我们通常需要处理2个问题：
1. 实例化这个组件以便调试
2. 为这个组件编写使用文档以便更好的让别人知道怎么使用这个组件

最原始的方法莫过于开发时建一个页面用于调试，开发完后再为其手写文档。然而一个详细的React组件文档应该包括：
1. 为各种使用场景编写demo以及对应的说明，同时附上demo的源码
2. 有demo可以当场体验而不是使用者要自己写代码后才能体验这个组件
3. 它的属性列表(`propTypes`)
4. 它的实例方法列表

如果你想做到以上估计得花上你一天的功夫，我希望能把精力放在开发更好的组件上剩下的能毫不费劲的优雅完成，于是我做了本文的主角[Redemo](https://github.com/gwuhaolin/redemo)。
Redemo是用来简单优雅的完成以上问题让你专注于开发自己的组件，剩下的一切它都为你做好了。先看下Redemo为组件生成文档的效果图或直接体验部分实践中的项目[redemo文档](https://gwuhaolin.github.io/redemo/)、[imuix](http://imweb.github.io/imuix/)：
![redemo效果图](https://github.com/gwuhaolin/redemo/raw/master/src/doc/redemo.png)
结构如下：
- 最上面是可立即体验的组件demo，同时可以用在开发过程中调试组件
- 组件实例下是这个demo的说明，支持markdown
- 接下来是组件的属性列表(`propTypes`)，支持markdown
- 最后是这个demo的源码


为你的组件生成这个你几乎不用写超过10行简单的代码更不用单独为组件写文档。假设你编写了一个Button组件，让我们来为Button组件编写一个demo：
1. 通过`npm i redemo` 安装 `redemo` 
2. 写下这些简单的代码
```js
import Redemo from 'redemo';
import Demo from './demo';//为一个使用场景实例化Button组件的demo源码
// 使用docgen 从 Button 组件源码里分析出 propTypes
const docgen = require('!!docgen-loader!../button');
// 读取为Button组件编写的demo的源码
const code = require('!!raw-loader!../demo');
const doc = `为这个demo做一些说明，支持*markdown*`
render(
<Redemo
  docgen={docgen}
  doc={doc}
  code={code}
>
  <Demo/>
</Redemo>
)
```

聪明的你大概会问以上代码并没有为Button属性编写文档，属性列表里的说明是哪来的？其实是通过[react-docgen](https://github.com/reactjs/react-docgen)从Button组件源码里提取出来的。大家都知道为代码写注释是个好习惯方便维护和理解，而这些注释正好也可以放在文档里一举两得。所以你在编写Button组件时需要为`propTypes`写注释，就像这样：
```js
class Button extends Component {
  static propTypes = {
    /**
     * call after button is clicked，支持*markdown*
     */
    onClick: PropTypes.func,
  }
  ...
}
```

想更深的了解redemo请看[这里](https://github.com/gwuhaolin/redemo)
希望redemo可以提升你的效率，觉得有用可告诉你的朋友。

[阅读原文](http://wuhaolin.cn/2017/04/30/%E5%BF%AB%E9%80%9F%E4%BC%98%E9%9B%85%E7%9A%84%E4%B8%BAReact%E7%BB%84%E4%BB%B6%E7%94%9F%E6%88%90%E6%96%87%E6%A1%A3/)