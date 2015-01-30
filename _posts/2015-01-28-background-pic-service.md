---
layout: post
title: "图床的诞生始末"
description: ""
category: "web"
tags: [ web, scala, product]
published: true
---

差不多花了一个月20天，我做了一个["无限流量"的图站](http://lelouchcrgallery.tk/)。但其实，我最最开始想做的并不是这个 ＝ ＝。但还是记录下，他是怎么一步一步进化到现在这个姿态的。


### 初衷

其实一开始我只是想找个项目练手scala，然后那一次正好在看 Ehentai 上的 cosplay ，由于echentai 服务器在国外，电信连过去很慢。于是我想做一个东西，能在晚上用电信很顺畅的看。

思路很简单，爬图，然后上传到网上找的国内图床（微博图床 和 贴图库）。然后就没了。

ehentai的爬取直接用了[现成的](https://github.com/4pr0n/ripme)，于是花了一个礼拜的晚上，就做完了，爬取部分。 

然后花了一天，研究了下 scalatra 做了个最最简单的web页面。于是发到了bandwagonghost。

可是，当天我还觉得挺有意思，但是之后我就不想看了。太无聊的东西了。于是一期落幕了。


### 图库

之后休息大概一个礼拜，主要在看[twitter scala school](http://twitter.github.io/scala_school/zh_cn/)和 [effective scala](http://twitter.github.io/effectivescala/index-cn.html) 玩了下scala。

直到某天在v2上看到了某人发了个帖子，做了个自动替换壁纸的脚本。

于是我就想到了个点子，做一个可以提供随机图作为背景图片的服务。给我的blog用。（表示之前是手工弄了3张图片，然后随机取一张作为背景图）

于是就有了现在[这个](http://lelouchcrgallery.tk/api)

api的设计参考了[hitokoto](http://hitokoto.us/api.html)。

最后，花了一个多月，陆陆续续做了4个图床的模拟上传。

找了2个资源比较不错的图源 p站和 yande。 yande 图多，可惜h也多 ＝ ＝。

最后由于压图需要很多内存 ＝ ＝ （weibo最大支持10mb的图），于是把爬单独部署在了bandwagong，web部分部署在了linode。

最后，其实图库只是个衍生品，啊哈哈。

好水的一篇博。。。
