---
layout: post
title: "来用console写字和画画"
description: ""
category: "linux"
tags: [linux,console,toilet,figlet,python]
---

之前有和同事讨论，redis或者rabbitmq启动的时候的画都是怎么画出来的，比如这个[redis](https://github.com/antirez/redis/blob/unstable/src/asciilogo.h)，感觉就是手打的一样。

google各种搜索后，发现些相关项目

###用console画ascii text

[toilet](http://caca.zoy.org/wiki/toilet)虽然名字确实有点。。。
以及他的宿主项目
[figlet](http://www.figlet.org/figlet-man.html)

figlet提供了一个font database，功能仅仅是将console的字体输出成该font的样子，比如

```sh
~/fonts $ toilet -f big.flf -t crawler
                         _
                        | |
  ___ _ __ __ ___      _| | ___ _ __
 / __| '__/ _` \ \ /\ / / |/ _ \ '__|
| (__| | | (_| |\ V  V /| |  __/ |
 \___|_|  \__,_| \_/\_/ |_|\___|_|
```
字体是从figlet下载的，装toilet时，默认也会自带一些字体库，我是用brew装的，默认字体库包含

```sh
~/fonts $ ls /usr/local/Cellar/toilet/0.3/share/figlet/
ascii12.tlf	bigascii9.tlf	circle.tlf	future.tlf	mono9.tlf	smascii9.tlf	smmono12.tlf
ascii9.tlf	bigmono12.tlf	emboss.tlf	letter.tlf	pagga.tlf	smblock.tlf	smmono9.tlf
bigascii12.tlf	bigmono9.tlf	emboss2.tlf	mono12.tlf	smascii12.tlf	smbraille.tlf	wideterm.tlf
```
想要别的库可以去[filglet font database](http://www.figlet.org/examples.html)挑选自己喜欢的。

###用console画ascii picture

[jp2a](http://csl.name/jp2a/) is a small utility that converts JPG images to ASCII.

这也是最最简单的画法了。

当然也可以自己画，参考了[果壳上的代码](https://github.com/aisk/pic2ascii/blob/master/pic2ascii.py)

然后发现，python的PIL库以及4年多没更新了，甚至不能用pip来装。。。

建议用这个[Pillow](http://python-imaging.github.io/)来替代PIL

还有将开头的代码改掉
```py
from PIL import Image
```

mac平台xcode升级了5.1的童鞋，
遇到`unused-command-line-argument-hard-error`的请继续[如此](http://stackoverflow.com/questions/22352838/ruby-gem-install-json-fails-on-mavericks-and-xcode-5-1-unknown-argument-mul)。哎，恼人的5.1 !!!



###### reference
http://www.guokr.com/blog/43470/
http://www.cnblogs.com/sukai/archive/2013/06/08/3127031.html

