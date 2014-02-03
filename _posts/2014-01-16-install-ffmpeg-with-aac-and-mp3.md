---
layout: post
title: "install ffmpeg with aac and mp3"
description: ""
category: "work"
tags: [ffmpeg,ubuntu]
---
{% include JB/setup %}

####本来
我之前一直很喜欢ubuntu，甚至觉得他比redhat系列更好用。因为我已经习惯使用apt命令了。直到。。。

####直到
最近我把所有爬虫的机器都迁到linux上面去。自然就先装上ubuntu server版本（我装的时12.04LTS版本）。32位的破机器都很稳定，装jvm，交叉便宜ffmpeg等都很顺利。可是再装最后一台也是唯一的64位的机器时却发生了令人意想不到的事情。在交叉编译完ffmpeg之后，兴冲冲地测试下转码时，在转aac时，竟然报错了，错误很简单，也很令人无语。

	segmentfault，core dumped

这尼玛遇到null point了，怎么这种事情都会发生？然后各种google不到。
明明线上的ffmpeg都是好的（线上的是centos）

绝望之余，便出下策想把ffmpeg源码check下来，gdb看看到底哪里出了问题，后来搜时都到一条命令
	
	dmesg --dmesg命令，找到最近发生的段错误输出信息

段错误上面一排libfftw3.so error
google一搜，https://trac.ffmpeg.org/ticket/943
果然有人和我一样遇到了这个问题。
把libfftw3-dev apt remove掉，重新编译，问题解决。

其实把libaacplus换成libfaac 时可以用的，不过faac编码出来的文件大小比libaacplus大了一倍有余，略坑。。

####现在
嗯哼，珍爱生命，远离ubuntu。话说我已经把自己的ubuntu卸了，换fedora了。（不过我可能又走上了另一条不归路）


