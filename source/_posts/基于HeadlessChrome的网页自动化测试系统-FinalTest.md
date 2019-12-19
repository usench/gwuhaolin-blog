---
title: 基于HeadlessChrome的网页自动化测试系统-FinalTest
date: 2017-06-26T02:59:41Z
url: https://github.com/gwuhaolin/blog/issues/7
tags:
    - chrome
---

**本文还未完成，还在不断补充中**

> 你的团队是不是遇到过网页出故障了无法第一时间知道只能等用户反馈后才知道出现了问题？我曾因为一次疏忽导致产品的一个页面不可用被批评，这让我难受低沉了一个星期。

我总是追求合理高效的方法来解决我所遇到的问题，于是我幻想要是有一个工具可以智能的监视网页一旦出现异常就提醒我们该多好。就在出故障的几天后chrome团队宣布chrome[支持headless模式](https://developers.google.com/web/updates/2017/04/headless-chrome)，这让我很兴奋因为它正是我在找的，于是我开始做[FinalTest]()（等成熟了再开源）。

# HeadlessChrome介绍
HeadlessChrome支持chrome所具有的所有功能只不过因为不显示界面而更快资源占用更小。相比于之前的[phantomjs](http://phantomjs.org)(作者因为HeadlessChrome的推出而宣布停止维护)chrome的优势在于它又一个很强的爹(google)会一直维护它优化它，并且chrome在用户量、体验、速度、稳定性都是第一的，所以我认为HeadlessChrome会渐渐替代之前所有的HeadlessBrowser方案。

### 如何操控HeadlessChrome
既然HeadlessChrome是以无界面模式运行的，那要怎么控制它和它交互？
chrome提供了远程控制接口，目前可以通过[chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface)来用js代码向chrome发送命令进行交互。在启动chrome的时候要开启远程控制接口，然后通过 chrome-remote-interface 连接到chrome后再通过协议控制chrome。具体操作见文档：
- [以headless模式和远程控制模式启动chrome](https://developers.google.com/web/updates/2017/04/headless-chrome)
- [连接到远程chrome控制它](https://github.com/cyrus-and/chrome-remote-interface#sample-api-usage)
- [控制chrome时支持哪些操作具体怎么用](https://chromedevtools.github.io/devtools-protocol/)


# FinalTest介绍
FinalTest是一个自动化的网页功能异常检测工具。名称来源于它做了测试阶段的最后一环，也是最后一道质量保障。

### FinalTest目标
- 自己会定时去检测出网页的功能异常，第一时间通知相关人
- 不需要对原网页产生任何改变就能使用
- 任何类型的网页都可以接入，包括服务器渲染的和浏览器渲染的
- 无需做太多工作就可接入已有网页，即便对于复杂场景要写单元测试注入运行，写单测的过程也是很轻松可调试的

需要特别说明的是FinalTest只是用来检查网页的功能异常（无法正常使用）而不做浏览器兼容性检查。

### 如何检测网页功能异常
当发生以下情况时，FinalTest会认为网页是异常的：
- 网页加载失败（后端服务挂了）
- 网页运行时出现未捕获的异常（前端js有问题）
- 控制台输出了error或warn日志（前端js有问题）
- 网络请求异常（CDN出问题，文件丢失），或者请求成功但是返回结果异常（后端服务有问题）
- 对于复杂的情况，可以为网页写测试脚本注入到浏览器里运行（一些交互性的功能无法使用）

### FinalTest架构
![FinalTest架构图](https://user-images.githubusercontent.com/5773264/27568503-cafec6b6-5b24-11e7-8855-79e85d0b6576.png)

其中包含这些模块：
- [chrome-runner](https://github.com/gwuhaolin/chrome-runner) 以控制模式启动系统上的chrome，并且守护着它保证chrome一直处于正常运行状态
- [chrome-pool](https://github.com/gwuhaolin/chrome-pool) 来自于数据库连接池的概念，管理chrome tabs重复利用tab提高chrome执行效率减少资源占用
- [chrome-tester](https://github.com/gwuhaolin/chrome-tester) 检测执行者，控制chrome运行网页找出出行的异常
- 管理后台 是用与接入网页时录入和实时查看监测运行状态的网页，和录入警告相关人
- reporter-web 是用于收集chrome-tester跑出的结果整理后通过[socket.io](https://socket.io/)传输给管理后台展示，SPA网页应用
- reporter-notice 是用于收集chrome-tester跑出的结果判断是否要警告，发生警告给相关人

### FinalTest难点
- 部分页面依赖登入态，如何简单的保证在测试一个页面前有正确的登入态？虽然通过CGI模拟登入或者控制浏览器完成登入步骤是可行的，但是这样流程是不是有点繁琐？
- 部分复杂功能可能需要进行一系列操作后才会进入，要测试这些功能目前好像只能通过写e2e测试脚本注入到浏览器里运行，但是写e2e测试会增加开发成本和维护成本。
- 是不是可以跟踪页面dom的diff变化来作出异常判断？

### 与其它类似方案对比
- [Nightwatch.js](http://nightwatchjs.org)：采用Selenium控制phantomjs浏览器执行e2e测试，安装复杂执行效率低占用资源大，只适合本地开发时跑测试，不适合在服务器端批量高频运行。

[阅读原文](http://wuhaolin.cn/2017/06/26/%E5%9F%BA%E4%BA%8EHeadlessChrome%E7%9A%84%E7%BD%91%E9%A1%B5%E8%87%AA%E5%8A%A8%E5%8C%96%E6%B5%8B%E8%AF%95%E7%B3%BB%E7%BB%9F-FinalTest/)