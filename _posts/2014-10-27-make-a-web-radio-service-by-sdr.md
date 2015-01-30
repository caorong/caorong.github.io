---
layout: post
title: "如何用 sdr 做一个在线收音机服务"
description: ""
category: "sdr"
tags: [ web, sdr, gnuradio, ffmpeg]
published: true
---

最近一直在做类似 qingting.fm， anyradio.cn 的直播。

本来想的很简单，他们一定是爬直播流，比如 `www.radio.cn` 上的。然后发现这个网上的上海电台的地址都挂了，再然后 qingting.fm 上的直播流都可以播放。。。

要么就是他们由电台独家给的地址，要么他们是直接收集的无线电信号？

那么如果要收集无线信号并能将数据传到 pc 用什么设备？

sdr（hackrf，rtl-sdr）光做收音，后者便宜45rmb

那么如何控制想要接受的频段，以及如何将raw audio 输出？

用 gnuradio 写代码，拖控件。

[gnuradio 的安装]()

步骤如下 

1. 实现能够接收fm

具体参考 hackrf官方 tutorial 1 https://greatscottgadgets.com/sdr/1/

2. 实现一个程序能同时接收多个频道 

参考 tutorial 1的课后习题 https://greatscottgadgets.com/sdr/2/  [flowgraph for lesson 2 homework](https://greatscottgadgets.com/sdr/grc/lesson2.grc)

3. 实现对外将多个频道通过udp 或者 tcp 传出去

路线图

hackrfSDR -> computer -> wav

computer 主要做了修改采样频率，最后， 将模拟信号强转成short（16bit）然后将raw audio 通过 udp 发出去， 然后ffmpeg 爬

[grc代码](https://github.com/caorong/testgrc)

参考里面的 multifm_noGUI.grc 

---

15.1.30

最后直播没有选择这个备选方案 ＝ ＝

表示这篇烂尾文，一直拖到现在，因为中间太忙了，还去研究了卫星锅爬卫星广播信号。也是醉了。

主要原因是去参加了pycon 遇到了蜻蜓的小伙伴。于是知道他们是纯网络流。好吧，我想多了。

