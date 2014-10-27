---
layout: post
title: "如何用 sdr 做一个在线收音机服务"
description: ""
category: "sdr, gnuradio, ffmpeg"
tags: [ web, sdr, gnuradio, ffmpeg]
published: false
---

最近一直在做类似 qingting.fm， anyradio.cn 的直播。

本来想的很简单，他们一定是爬直播流，比如 `www.radio.cn` 上的。然后发现这个网上的上海电台的地址都挂了，再然后 qingting.fm 上的直播流都可以播放。。。

要么就是他们由电台独家给的地址，要么他们是直接收集的无线电信号？

于是网上搜索一番确实找到了点思路。

原理是用 sdr 连接 pc 然后用 gnuradio 将逻辑信号通过 udp 或者 tcp 服务发出去。

gnuradio 线路图如下










