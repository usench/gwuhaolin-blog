---
title: 使用HeadlessChrome做单页应用SEO
date: 2017-06-27T11:59:22Z
url: https://github.com/gwuhaolin/blog/issues/8
tags:
    - chrome
---

> 随着react、vue、angular等前端框架的流行越来越多的web应用变成了单页应用，它们的特点是异步拉取数据在浏览器中渲染出HTML。使用这些框架极大的提升web用户体验和开发效率的同时缺带来一个新问题，那就是这样的网页无法被搜索引擎收录。虽然这些web框架支持服务端渲染，但这可能又会增加开发成本。

有没有一个可用于任何单页应用的SEO解决方案，让我们不用对代码做改变保持原有的开发效率？[chrome-render](https://github.com/gwuhaolin/chrome-render)可以帮我们做到这点，它通过控制HeadlessChrome渲染出最终的HTML返回给爬虫来实现。

# HeadlessChrome介绍
前不久chrome团队宣布chrome[支持headless模式](https://developers.google.com/web/updates/2017/04/headless-chrome)，HeadlessChrome支持chrome所具有的所有功能只不过因为不显示界面而更快资源占用更小。相比于之前的[phantomjs](http://phantomjs.org)(作者因为HeadlessChrome的推出而宣布停止维护)chrome的优势在于它又一个很强的爹(google)会一直维护它优化它，并且chrome在用户量、体验、速度、稳定性都是第一的，所以我认为HeadlessChrome会渐渐替代之前所有的HeadlessBrowser方案。

### 如何操控HeadlessChrome
既然HeadlessChrome是以无界面模式运行的，那要怎么控制它和它交互？
chrome提供了远程控制接口，目前可以通过[chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface)来用js代码向chrome发送命令进行交互。在启动chrome的时候要开启远程控制接口，然后通过 chrome-remote-interface 连接到chrome后再通过协议控制chrome。具体操作见文档：
- [以headless模式和远程控制模式启动chrome](https://developers.google.com/web/updates/2017/04/headless-chrome)
- [连接到远程chrome控制它](https://github.com/cyrus-and/chrome-remote-interface#sample-api-usage)
- [控制chrome时支持哪些操作具体怎么用](https://chromedevtools.github.io/devtools-protocol/)

# chrome-render原理与实践
### 原理
chrome-render先会通过[chrome-runner](https://github.com/gwuhaolin/chrome-runner)以headless模式启动和守护你操作上的chrome，再通过chrome-remote-interface操控chrome去访问需要被SEO的网页让chrome运行这个网页，等到包含数据的HTML被渲染出来时读取当前网页DOM转换成字符串后返回。

怎么知道你的网页什么时候已经渲染出包含数据的HTML了可以返回了呢？为了提升chrome-render效率，默认会在`domContentEventFired`时返回。对于复杂的场景还可以通过开启chrome-render的`useReady`选项，等到网页里调用了`window.chromeRenderReady()`时返回。

只渲染出了HTML还不够我们还需要检测出来着搜索引擎爬虫的访问，如果请求来着爬虫就返回chrome-render渲染后的HTML否则返回正常的单页应用所需HTML。

综上，整体架构如下：
![koa-seo arch](https://github.com/gwuhaolin/koa-seo/raw/master/doc/koa-seo%20arch.png)

### 实践
只需以下几行简单代码就可让chrome渲染出HTML：
```js
const ChromeRender = require('chrome-render');
ChromeRender.new().then(async(chromeRender)=>{
    const htmlString = await chromeRender.render({
       url: 'http://qq.com',
    });
});    
```
chrome-render只是做了渲染出HTML的工作，要实现SEO还需要和web服务器集成。为了方便大家使用我做了一个koa中间件[koa-seo](https://github.com/gwuhaolin/koa-seo)，要集成到你现有的项目很简单，如下：
```js
const seoMiddleware = require('koa-seo');
const app = new Koa();
app.use(seoMiddleware());
```
只需像这样接入一个中间件你的单页应用就被SEO了。

# 应用场景扩展
chrome-render除了用于通用SEO解决方案其实可以用于通用服务端渲染，因为目的都是渲染出最终的HTML再返回。针对通用服务端渲染我也做了一个koa中间件[koa-chrome-render](https://github.com/gwuhaolin/koa-chrome-render)。使用chrome-render做服务端渲染的

优势在于：
- 通用，适用于所有单页应用
- 对原有代码几乎无改动，最多再合适的地方加个`window.chromeRenderReady()`，保持原有开发效率

缺点在于：
- 和react、vue等只带的服务端渲染相比性能低（经我测试大约 200ms vs 60ms）
- chrome-render渲染时占用资源高，一次渲染大约占用25Mb内存，当请求量大时服务器可能扛不住。但是可以通过缓存渲染结果优化。

# 总结
大家可能会说这个很像[prerender.io](https://prerender.io)，没错思路是一样的，chrome-render的优势在于：
- chrome-render开源可自己部署，prerender要收费是商业产品
- prerender基于已经停止维护的phantomjs

本文中所提到的相关项目都是开源的并且有详细的使用文档，它们的文档链接如下：
- [chrome-render](https://github.com/gwuhaolin/chrome-render)
- [chrome-runner](https://github.com/gwuhaolin/chrome-runner)
- [koa-seo](https://github.com/gwuhaolin/koa-seo)
- [koa-chrome-render](https://github.com/gwuhaolin/koa-chrome-render)

喜欢的给个star，希望大家和我一起来改进它们让它们更强大。


[阅读原文](http://wuhaolin.cn/2017/06/27/%E4%BD%BF%E7%94%A8HeadlessChrome%E5%81%9A%E5%8D%95%E9%A1%B5%E5%BA%94%E7%94%A8SEO/)