---
layout: post
title: "制作raspberrypi image"
description: ""
category: "raspberrypi"
tags: [raspberrypi, hacking]
published: true
---


上个月再 RS 买了个树莓派， 等了一个月之后终于送了过来。记录安装一下过程。


去[这里](http://www.raspberrypi.org/downloads/)下载镜像



[官网的安装教程](http://www.raspberrypi.org/documentation/installation/installing-images/mac.md)

注意 如果按照教程的 `dd` 方式会非常慢

```sh
sudo dd bs=1m if=pi-snappy.img of=/dev/disk1
```

需要改成

```sh
sudo dd bs=1m if=pi-snappy.img of=/dev/rdisk1
```

原因：

	/dev/rdisk nodes are character-special devices, but are "raw" in the BSD sense and force block-aligned I/O. They are closer to the physical disk than the buffer cache. /dev/disk nodes, on the other hand, are buffered block-special devices and are used primarily by the kernel's filesystem code.

还有

`dd` 的时候可以按 `CTRL-T` 查看进度


启动后 密码注意查看[这里](http://www.raspberrypi.org/downloads/)

quick-start-guide 说登录密码是 pi/raspberry， 但如果你装的不是 NOOBS 密码要各自的 `Default Login`

```
Default login:  ubuntu / ubuntu
``` 

我装的是 ubuntu snappy, 启动后原来的 `apt-get` 由 `snappy` 替代了，然后初始时间是1970，需要改成当前的，不然 ssl 报错

```sh
sudo date 032320042015.00

#format is mm ddhhnnyyyy.ss, nn=minutes, so the 22th of march 2015, 23:04 sharp
```
http://www.raspberrypi.org/forums/viewtopic.php?f=56&t=98633


但是我那啃爹 snappy gcc, git, make, 除了python 别的什么都没有。snappy search 也什么都搜不到。

```
ubuntu@localhost:~$ sudo snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  2          3          f442b1d8d6db3f  *
webdm        edge  0.1        -          1604c8b7c9f6c5  *
```



于是放弃了ubuntu snappy，[改装了 ubuntu 14.04LTS](https://wiki.ubuntu.com/ARM/RaspberryPi)

装能装上去，但问题是，这个源没有国内镜像，一个 apt-update 都完成不了，速度只有1kb 卧槽了。


最后换成了 Debian Wheezy (推荐)。

然后ssh进入后，开头由下面这东东 

```text
NOTICE: the software on this Raspberry Pi has not been fully configured. Please run 'sudo raspi-config'
```

最后跟着初始设置就行了。


#### reference

http://www.raspberrypi.org/forums/viewtopic.php?f=2&t=3360

http://daoyuan.li/solution-dd-too-slow-on-mac-os-x/

http://superuser.com/questions/631592/mac-osx-why-is-dev-rdisk-20-times-faster-than-dev-disk

http://raspberrypi.stackexchange.com/questions/499/how-can-i-resize-my-root-partition








