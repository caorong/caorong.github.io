---
layout: post
title: "udp 替代 tcp？"
description: ""
category: "network"
tags: [QUIC, tcp, udp, kcp]
published: true
---

话说最近看到 2 个消息，google 将使用 QUIC 协议，ss-android 还看到个 changelog 开始支持kcp。

于是就查了下资料看看, 发现其实 QUIC 和 kcp 这2东东其实都是为类似的目的开发的类似的东西。

首先 他们 是基于 UDP 的自定义的协议。

**那么为什么要自定并用 UDP?**

他们都为了能在不修改系统层的情况下，解决 tcp 丢包重穿带来的延迟的问题。

**用了什么策略?**

策略可以参考kcp 的[技术特性](https://github.com/skywind3000/kcp#技术特性)

**结论，他们其实都为了改善 `高丢包` 环境的网络质量。**

QUIC 的目的是为了在2g，3g下，目前也只有google自己的网站支持。

而 kcp 则是为了fq的目的，解决自己的境外vps 与 国内的网络质量。


再扯一下QQ， 貌似用 基于 udp 来自定义tcp 的事情 [很早之前qq就是这么做的, 虽然初衷并不同](https://www.zhihu.com/question/20292749)


不过对于 自己的vps来说我们有能力自己sudo，所以其实很早就有类似 锐速 这种基于tcp的加速软件。还有类似原理[开源版本](https://github.com/snooda/net-speeder.git)

不过 这种基于 tcp 的方式，原理都是靠发多倍的数据包，减小丢包的情况。不过如果人人都这么搞，会让原本已经拥塞的国际网络出口更加堵。


## 网络基础

为了理解kcp 的配置方法，需要再复习下网络基础

网络按照 5 层来划分的话，可以分为

1. 物理层
2. 数据链路层
3. 网络层
4. 运输层
5. 应用层

#### 物理层

建立在物理通信介质的基础上，作为系统和通信介质的接口，用来实现数据链路实体间透明的比特 (bit) 流传输。只有该层为真实物理通信，其它各层为虚拟通信，这里并不关心

#### 数据链路层

数据链路层的信道主要有两种 `点对点` 和 `广播`， 前者主要是一对一的通信，后者用于局域网通信, 不做展开

点对点通信通过差错控制提供数据帧（Frame）在信道上无差错的传输。（帧有check sum）

先说下 点对点信道 的数据链路层 的协议数据单位 － `帧`，而这层不同的协议，都有不同的 `最大传输单元 MTU`, 所谓的 MTU 就是帧的数据部分的数据大小。

目前链路层协议一般用点对点协议 ppp (Point-to-point Protocol)，所以上层网络层的数据包将被组装成 ppp 协议的帧。

ppp 协议的帧格式 可以参考[百科](http://baike.baidu.com/subview/30514/8100295.htm?fromtitle=PPP%E5%8D%8F%E8%AE%AE&fromid=1988803&type=syn)

最重要的是，一般 ppp 帧的 MTU `不超过 1500 字节`。

所以尽量提升MTU 的大小可以减少拆包的个数，可以提升速度。

mac 上可以通过  `ping -D -s 1414 45.63.xx.xx` 来测试最大的MTU 大小。

我的 VPS 可接受 `MTU:1500`, 路由器 能达到 1472，不过 vps 的 MTU 实测下来 只能达到 1414, mtr下，发现因为被我的 ISP (222.72.255.206) 坑了。

#### 网络层

选择合适的路由，尽最大努力使数据分组（packet）交付到目的主机，不提供服务质量。所传的包可错，可丢，可乱序，可重复。

#### 运输层

主要就是 tcp， udp 2个协议, 也是配置kcp 参数需要重点理解的

tcp： 面向连接， 可靠的流协议。

udp： 之保证发了消息，别的不保证。

也就是说，tcp 在udp 之上增加了许多功能，以保证他的可靠，所以，他的头部也比 udp 大很多，具体可以参考[这里 ](http://www.jianshu.com/p/8be9b3204864) 

所以 QUIC 和 kcp 为了实现可靠性，顺序性，所以在udp 的数据包内，加入了很多类似tcp的功能，虽然她们的头会比tcp的小，但不会小太多。

所以 kcp 也实现了 ARQ，滑动窗口等东西。

所以，如果网络质量实在太差，即使 用了 kcp 减少了延迟，但还是会卡，只不过可能比tcp 好一些。

kcp 具体的技术特性[参考官方文档](https://github.com/skywind3000/kcp#技术特性)

##### 拥塞控制 

对于tcp 来说 ，由于需要考虑拥塞控制和流量控制，这两方面的内容，发送端的发送窗口为min(cwnd，rwnd)，但是rwnd是由对端确定的，网络环境对其没有影响，所以在考虑拥塞的时候我们一般不考虑rwnd的值，我们暂时只讨论如何确定cwnd值的大小。

在执行慢开始算法时，拥塞窗口 cwnd的初始值为 1，发送第一个报文段。当发送端收到来自接收端的ACK之后，拥塞窗口开始以1、2、4这样的指数形式增长。当拥塞窗口cwnd 增长到慢开始门限值 ssthresh 时，就改为执行拥塞避免算法，拥塞窗口按线性规律增长。

拥塞避免：

最初，拥塞窗口指数增长，可以很快进行大数据的发送，最大限度的利用网络宽带资源。当达到慢开始门限值，开始进入拥塞避免阶段，拥塞窗口开始加法增加。这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。

也就是说，tcp 的发送窗口会根据发送情况，不断的 变大变小，(丢包的话就会变小)

而对于 kcp 来说，他不能自动的，需要自己设定 client rcvwnd和server sndwnd。如果过大，反而会造成人为造成的丢包。

调整方式参考[文档](https://github.com/xtaci/kcptun#推荐参数-lollipop)

所以，如果个人的网络质量比较好，比如有接入国际精品网(基本上 丢包率0%)，那还是用tcp吧, 多省事。

##### tcp 拥塞算法

对于tcp 协议，还可以 修改更合适的TCP拥塞控制算法。

关于tcp 拥塞算法的介绍 [参考这里](http://blog.csdn.net/zhangskd/article/details/6715751) 和 [wiki](https://en.wikipedia.org/wiki/TCP_congestion_control) 

这里主要说2个

1. 专门为卫星通讯设计的拥塞控制算法：Hybla, 适合高延迟高丢包的美国vps
2. 适合高带宽，高延迟的网络拥塞控制算法：HTCP, 比如日本vps


通过以下命令查看本机支持的拥塞算法模块

```
sysctl net.ipv4.tcp_available_congestion_control
```

如果没有 htcp， hybla 算法, 可以尝试通过modprobe启用模块

```
/sbin/modprobe tcp_htcp
/sbin/modprobe tcp_hybla
```

网上有很多教程都是针对高并发的，我不关注，对于高丢包高延迟的情况我们通过增大缓存来提高TCP性能

/etc/sysctl.conf

```
# increase TCP max buffer size settable using setsockopt()
net.core.rmem_max = 67108864 
net.core.wmem_max = 67108864 
# increase Linux autotuning TCP buffer limit
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
# increase the length of the processor input queue
net.core.netdev_max_backlog = 250000
# recommended for hosts with jumbo frames enabled
net.ipv4.tcp_mtu_probing=1

net.ipv4.tcp_congestion_control=hybla
```

对于丢包少，高延迟网络，建议使用htcp


```
net.ipv4.tcp_congestion_control=htcp
```

记得设置完后执行 

```
sysctl -p
```

## 实测结果

由于我家是国际精品网，所以 测试下来 kcp 对我来说没什么变化。

不过 在启用 htcp 后，下载速度能有明显的提升, 以下分别是 使用 cubic 和 htcp 的speed test 结果


```
~ ᐅ proxychains4 wget 'http://speedtest.singapore.linode.com/100MB-singapore.bin'
[proxychains] config file found: /usr/local/Cellar/proxychains-ng/4.8.1/etc/proxychains.conf
[proxychains] preloading /usr/local/Cellar/proxychains-ng/4.8.1/lib/libproxychains4.dylib
[proxychains] DLL init: proxychains-ng 4.8.1
--2016-08-14 19:19:00--  http://speedtest.singapore.linode.com/100MB-singapore.bin
Resolving speedtest.singapore.linode.com... 224.0.0.1
Connecting to speedtest.singapore.linode.com|224.0.0.1|:80... [proxychains] Dynamic chain  ...  127.0.0.1:1080  ...  speedtest.singapore.linode.com:80  ...  OK
connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/octet-stream]
Saving to: '100MB-singapore.bin'

100MB-singapore.bin                      23%[==================>                                                              ]  23.66M  2.49MB/s   eta 30s   ^C
~ ᐅ proxychains4 wget 'http://speedtest.singapore.linode.com/100MB-singapore.bin'
~ ᐅ rm 100MB-singapore.bin
~ ᐅ proxychains4 wget 'http://speedtest.singapore.linode.com/100MB-singapore.bin'
[proxychains] config file found: /usr/local/Cellar/proxychains-ng/4.8.1/etc/proxychains.conf
[proxychains] preloading /usr/local/Cellar/proxychains-ng/4.8.1/lib/libproxychains4.dylib
[proxychains] DLL init: proxychains-ng 4.8.1
--2016-08-14 19:19:24--  http://speedtest.singapore.linode.com/100MB-singapore.bin
Resolving speedtest.singapore.linode.com... 224.0.0.1
Connecting to speedtest.singapore.linode.com|224.0.0.1|:80... [proxychains] Dynamic chain  ...  127.0.0.1:1080  ...  speedtest.singapore.linode.com:80  ...  OK
connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/octet-stream]
Saving to: '100MB-singapore.bin'

100MB-singapore.bin                      30%[=======================>                                                         ]  30.15M  6.78MB/s   eta 20s   ^C

```

## 结论

对于 bwg 这种 openvz 的机器，由于不能 添加tcp 算法模块，所以建议使用 [kcptun](https://github.com/xtaci/kcptun) 

对于 vultr 这种 日本 机器，根据丢包情况可以测试下 修改tcp 算法 或者使用 kcp 的效果，选效果更好的。

对于 不丢包的情况，强烈建议 直接tcp，并使用 htcp 算法。

# reference

https://www.zhihu.com/question/29705994

http://bobao.360.cn/news/detail/3399.html

http://www.jianshu.com/p/8be9b3204864

http://www.jianshu.com/p/eab86c0d1612

https://hong.im/2013/04/20/linux-tcp-tuning/

http://blog.csdn.net/zhangskd/article/details/6715751

http://www.weeiy.com/linux-tcp-hybla.html

[大学时教材，计算机网络](https://book.douban.com/subject/2970300/)

http://coolshell.cn/articles/11609.html
