---
layout: post
title: "在mac上编译安装quack3"
description: ""
category: "game"
tags: [game,quack3,mac]
---
 

首先，这是github上id的[官方源码](https://github.com/id-Software/Quake-III-Arena),可以在win上用vc， linux上gcc等编译，但在mac上用xcode会提示说此项目太古老。。。

于是google上便找到了这个[非官方源码](https://github.com/ioquake/ioq3)
这个非官方组织在q3的基础上继续维护q3。

安装方法：

1. 我是64位的系统，于是console下直接 `./make-macosx.sh x86_64`
2. 此时编译完后在build/releasexx/ 文件夹下直接运行会提示错误。因为仅仅编译了引擎，而没有资源文件-pak0.pk3。下载后将他放在ioquake3.app里面的ioquake3.app/MacOs/baseq3 directory.[资源文件地址](http://pan.baidu.com/s/1dDqrMtb)
3. 下载资源文件[更新包](http://ioquake3.org/extras/patch-data/)同样合并到oquake3.app/MacOs/下。
4. play it with fun～