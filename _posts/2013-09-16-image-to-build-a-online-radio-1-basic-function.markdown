---
layout: post
title: "如何做一个基于hls的网络收音机（一）基本功能"
tagline: "yy一个qingting.fm"
description: "-desc-"
category: web
tags: [YY, hls, web]
---

一个类似qingting.fm是如何实现的。纯属个人yy，请勿较真。

#####首先，用hls相比以前ms的mms或者开源的rtmp流做网络收音机有什么好处？

其实网上有许多现成的基于rtsp，rtmp，mms，mmsh的国家，地方或私有网络电台，google可以搜出来很多网站，可以直接用他们的流播放呀，为啥要费服务器资源再去搭建个基于hls的电台呢？

1. 可以承受高并发    **（主要原因）**
2. 不需要特殊服务器，成本低
3. 可以回放，点播  **（顺便实现）**

而对于当今的互联网来说，上网的人越来越多，如果用以前的保持长链接的red5server来建集群承受并发实在是成本太高。

#####如何做一个自己的hls子文件（.m3u8 + .ts）？

其实网上有很多opensource项目都帮你实现了子功能。
这里直接用[ffmpeg](http://ffmpeg.org/)和m[3u8segment](https://github.com/johnf/m3u8-segmenter)


`ffmpeg -er 4 -i mmsh://203.141.56.46:80/BAN-BAN_Radio?MSWMExt=.asf test.mp3 -f mpegts -acodec libmp3lame -ar 22050 -ab 32k -vn - | \`
 `m3u8-segmenter -i - -d 7 -p testdir/bbradio -m testdir/bb.m3u8 -u http://192.168.xx.xx/
`

然后架设一个nginx，将静态文件目录指向testdir，改好-u的ip地址，然后ffplay -i 192.168.xx.xx/bb.m3u8 or 用vlc等进行test。

##### final

如果想做出一个类似qingtimg.fm类似的demo（一个最多相差7s的实时的广播）可以在再启一个服务，这个服务根据currenttime-configtime酸楚一个bbradio-xx的xx数字，然后分别将xx,xx+1,xx+2 ，3行写入m3u8文件，return给client。 由此便实现了个对于用户来说时间差最多相差7秒的适时online radio～

 