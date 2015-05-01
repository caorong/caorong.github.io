---
layout: post
title: "Ubuntu 安装黑屏"
description: ""
category: "ubuntu"
tags: [ubuntu, life, ubuntu]
published: true
---

## 异常蛋疼的装 ubuntu，win7 双系统经历

## 环境

新组装机， i7, N卡

## 首次安装进入 install ubuntu 后黑屏

装玩 windows 后，重启，更改bios的启动顺序，重启，插入ubuntu 的 usb installer 

然后进入选择界面，选择 `install ubuntu` 后黑屏。

### [解决方法](http://wiki.ubuntu-tw.org/index.php?title=FAQinstall)

在选择界面，按e ，修改开机选项， 然后在 linux 一行，找到"quite splash" 然后在后面加上 `nomodeset`

然后就能进入安装了。

## 安装完后重启之后没看到grub直接黑屏

这说明了2件事情

1. grub没看到 说明win7的分区信息消失了，只剩下了默认的ubuntu 分区，于是没有显示grub，而直接进入ubuntu了。

2. 进入系统黑屏，遇上吗一样的情况 n卡的问题

### 解决方法

1想办法进入系统重写grub

1. 用livecd进入系统

我先装的 win7 再装的 ubuntu，所以我的sda1是win7 sda3 是ubuntu

```text
sudo mount /dev/sda3 /mnt

## 表示硬盘
sudo grub-install --root-directory=/mnt /dev/sda
```

2. 按照下面方式直接进入


2的解决方法同上。不过要想办法强制进入grub，开机时按住`shift`不放即可进入grub，然后同上增加`nomodeset`


最后进入 ubuntu 改grub， 添加 `nomodeset`

```
vim /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
## 改称
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
```


## 装 N卡驱动




## 总结

其实可以不需要再用live cd 模式进入 ubuntu 重写 grub， 只不过当时我以为是连ubuntu的分区信息都木有了，于是就用live cd 模式重写了grub。






#### reference

http://wiki.ubuntu-tw.org/index.php?title=FAQinstall

http://askubuntu.com/questions/207663/cannot-update-grub-with-paramters-on-live-usb

http://dectinc.cc/2014/07/13/ubuntu-14-04-black-screen/


