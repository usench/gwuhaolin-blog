---
title: 使用flv.js做直播
date: 2017-05-17T08:16:24Z
url: https://github.com/gwuhaolin/blog/issues/3
tags:
    - 音视频
---

为什么要在这个时候探索flv.js做直播呢？原因在于各大浏览器厂商已经默认禁用Flash，之前常见的Flash直播方案需要用户同意使用Flash后才可以正常使用直播功能，这样的用户体验很致命。

在介绍flv.js之前先介绍下常见的直播协议以及给出我对它们的延迟与性能所做的测试得出的数据。
如果你看的很吃力可以先了解下音视频技术的一些[基础概念](https://github.com/gwuhaolin/blog/issues/5)。


## 常见直播协议
- **RTMP**: 底层基于`TCP`，在浏览器端依赖Flash。
- **HTTP-FLV**: 基于`HTTP`流式IO传输FLV，依赖浏览器支持播放FLV。
- **WebSocket-FLV**: 基于`WebSocket`传输FLV，依赖浏览器支持播放FLV。`WebSocket`建立在`HTTP`之上，建立`WebSocket`连接前还要先建立`HTTP`连接。
- **HLS**: Http Live Streaming，苹果提出基于`HTTP`的流媒体传输协议。`HTML5`可以直接打开播放。
- **RTP**: 基于`UDP`，延迟1秒，浏览器不支持。

#### 常见直播协议延迟与性能数据<sub><sup>以下数据只做对比参考</sup></sub>
| 传输协议 | 播放器 | 延迟 | 内存 | CPU  |
|----------|--------|------|------|------|
| RTMP     | Flash  | 1s    | 430M | 11%  |
| HTTP-FLV | Video  | 1s    | 310M | 4.4% |
| HLS      | Video  | 20s   | 205M | 3%   | 

在支持浏览器的协议里，延迟排序是： 
**RTMP = HTTP-FLV = WebSocket-FLV  < HLS**
而性能排序恰好相反： 
**RTMP > HTTP-FLV = WebSocket-FLV  > HLS**
也就是说延迟小的性能不好。

可以看出在浏览器里做直播，使用HTTP-FLV协议是不错的，性能优于RTMP+Flash，延迟可以做到和RTMP+Flash一样甚至更好。

## flv.js 简介
flv.js是来自Bilibli的开源项目。它解析FLV文件喂给原生HTML5 Video标签播放音视频数据，使浏览器在不借助Flash的情况下播放FLV成为可能。

## flv.js 优势
- 由于浏览器对原生Video标签采用了硬件加速，性能很好，支持高清。
- 同时支持录播和直播
- 去掉对Flash的依赖

## flv.js 限制
- FLV里所包含的视频编码必须是`H.264`，音频编码必须是`AAC`或`MP3`， IE11和Edge浏览器不支持MP3音频编码，所以FLV里采用的编码最好是H.264+AAC，这个让音视频服务兼容不是问题。
- 对于录播，依赖 `原生HTML5 Video标签` 和 [Media Source Extensions](https://w3c.github.io/media-source/) API
- 对于直播，依赖录播所需要的播放技术，同时依赖 `HTTP FLV` 或者 `WebSocket` 中的一种协议来传输FLV。其中`HTTP FLV`需通过流式IO去拉取数据，支持流式IO的有[fetch](https://fetch.spec.whatwg.org)或者[stream](https://streams.spec.whatwg.org)
- ` flv.min.js` 文件大小 164Kb，gzip后 35.5Kb，flash播放器gzip后差不多也是这么大。
- 由于依赖`Media Source Extensions`，目前所有iOS和Android4.4.4以下里的浏览器都不支持，也就是说目前对于移动端flv.js基本是不能用的。

####  flv.js依赖的浏览器特性兼容列表
- [HTML5 Video](http://caniuse.com/#feat=webm)
- [Media Source Extensions](http://caniuse.com/#feat=mediasource)
- [WebSocket](http://caniuse.com/#feat=websockets)
- HTTP FLV: [fetch](http://caniuse.com/#feat=fetch) 或 [stream](http://caniuse.com/#feat=http-live-streaming)
## flv.js 原理
flv.js只做了一件事，在获取到FLV格式的音视频数据后通过原生的JS去解码FLV数据，再通过[Media Source Extensions](https://w3c.github.io/media-source/) API 喂给原生HTML5 Video标签。(HTML5 原生仅支持播放 mp4/webm 格式，不支持 FLV)

flv.js 为什么要绕一圈，从服务器获取FLV再解码转换后再喂给Video标签呢？原因如下：
1. 兼容目前的直播方案：目前大多数直播方案的音视频服务都是采用FLV容器格式传输音视频数据。
2. FLV容器格式相比于MP4格式更加简单，解析起来更快更方便。

## flv.js兼容方案
由于目前flv.js兼容性还不是很好，要用在产品中必要要兼顾到不支持flv.js的浏览器。兼容方案如下：
#### PC端
1. 优先使用 HTTP-FLV，因为它延迟小，性能也不差1080P都很流畅。
2. 不支持 flv.js 就使用 Flash播放器播 RTMP 流。Flash兼容性很好，但是性能差默认被很多浏览器禁用。
3. 不想用Flash兼容也可以用HLS，但是PC端只有Safari支持HLS
#### 移动端
1. 优先使用 HTTP-FLV，因为它延迟小，支持HTTP-FLV的设备性能运行 flv.js 足够了。
2. 不支持 flv.js 就使用 HLS，但是 HLS延迟非常大。
3. HLS 也不支持就没法直播了，因为移动端都不支持Flash。

## flv.js实战
说了这么多介绍与原理，接下来教大家如何用flv.js搭建一个完整的直播系统。
我已经搭建好了[一个demo](http://rawgit.com/gwuhaolin/reflv/gh-pages/live.html)可以供大家体验。

#### 搭建音视频服务
主播推流到音视频服务，音视频服务再转发给所有连接的客户端。为了让你快速搭建服务推荐我用go语言实现的[livego](https://github.com/gwuhaolin/livego)，因为它可以运行在任何操作系统上，对Golang感兴趣？请看[Golang 中文学习资料汇总](http://go.wuhaolin.cn/)。
1. [下载livego](https://github.com/gwuhaolin/livego/releases)，注意选对你的操作系统和位数。
2. 解压，执行`livego`，服务就启动好了。它会启动RTMP(1935端口)服务用于主播推流，以及HTTP-FLV(7001端口)服务用于播放。
#### 实现播放页
在react体系里使用react flv.js 组件[reflv](https://github.com/gwuhaolin/reflv) 快速实现。
先安装`npm i reflv`，再写代码：
```js
import React, { PureComponent } from 'react';
import Reflv from 'reflv';

export class HttpFlv extends PureComponent {
  render() {
    return (
      <Reflv
        url={`http://localhost:7001/live/test.flv`}
        type="flv"
        isLive
        cors
      />
    )
  }
}
```
让以上代码在浏览器里运行。这是你还看不到直播，是因为还没有主播推流。
- 你可以使用[OBS](https://obsproject.com)来推流，注意要配置好OBS：
<img width="961" alt="screen shot 2017-06-07 at 5 41 32 pm" src="https://user-images.githubusercontent.com/5773264/26872300-9c1c5bb4-4ba8-11e7-98eb-98e6d6ba751f.png">

- 也可以使用[ffmpeg](https://ffmpeg.org)来推流，推流命令`ffmpeg -f avfoundation -i "0" -vcodec h264 -acodec aac -f flv rtmp://localhost/live/test`

## flv.js延迟优化
按照上面的教程运行起来的直播延迟大概有3秒，经过优化可以到1秒。在教你怎么优化前先要介绍下直播运行流程：
1. 主播端在采集到一段时间的音视频原数据后，因为音视频原数据庞大需要先压缩数据：
    - 通过H264视频编码压缩数据数据
    - 通过PCM音频编码压缩音频AAC数据

2. 压缩完后再通过FLV容器格式封装压缩后的数据，封装成一个FLV TAG
3. 再把FLV TAG通过RTMP协议推流到音视频服务器，音视频服务器再从RTMP协议里解析出FLV TAG。
4. 音视频服务器再通过HTTP协议通过和浏览器建立的长链接流式把FLV TAG传给浏览器。
5. flv.js 获取FLV TAG后解析出压缩后的音视频数据喂给Video播放。

知道流程后我们就知道从哪入手优化了：
- 主播端采集时收集了一段时间的音视频原数据，它专业的叫法是[GOP](http://blog.csdn.net/zqj6893/article/details/50262189)。缩短这个收集时间(也就是减少GOP长度)可以优化延迟，但这样做的坏处是导致视频压缩率不高，传输效率低。
- 关闭音视频服务器的I桢缓存可以优化延迟，坏处是用户看到直播首屏的时间变大。
- 减少音视频服务器的buffer可以优化延迟，坏处是音视频服务器处理效率降低。
- 减少浏览器端flv.js的buffer可以优化延迟，坏处是浏览器端处理效率降低。
- 浏览器端开启flv.js的Worker，多线程运行flv.js提升解析速度可以优化延迟，这样做的flv.js配置代码是：
```js
{
          enableWorker: true,
          enableStashBuffer: false,
          stashInitialSize: 128,// 减少首桢显示等待时长
}
```
这里是[优化后的完整代码](https://github.com/gwuhaolin/reflv/blob/master/doc/live/http-flv.js)


[阅读原文](http://wuhaolin.cn/2017/05/17/%E4%BD%BF%E7%94%A8flv.js%E5%81%9A%E7%9B%B4%E6%92%AD/)