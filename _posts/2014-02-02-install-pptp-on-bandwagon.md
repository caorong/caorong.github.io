---
layout: post
title: "install pptp on bandwagon(openvz)"
description: ""
category: "life"
tags: [linux,pptp,gfw,vps]
---
 

####缘起
今天下午正好去办了网银，然后回家后看v2ex时正好看到搬瓦工的年付9.99刀的vps又出现了（话说之前还出了年付2.99刀的我又没有珍惜）512内存，5G硬盘，500G流量/per month。

####配置vpn
其实搬瓦工后台自带一键配置openvpn，不过考虑到之前openvpn正在被gfw各种针对的原因，还有之前听teahour讲云梯，感觉gfw应该不会对pptp下手，所以还是自己配置一个pptp的更靠谱些。

搬瓦工用的是不靠谱的openvz，系统：centos6 x86

`yum -y install make libpcap iptables gcc-c++ logrotate tar cpio perl pam tcp_wrappers dkms kernel_ppp_mppe ppp`

`uname -m `
查看系统64 or 32
然后选择合适版本的pptpd
比如我i686的

`wget http://poptop.sourceforge.net/yum/stable/rhel6Server/i386/pptpd-1.4.0-1.el6.i686.rpm`

`rpm -Uvh pptpd-1.4.0-1.el6.i686.rpm`

配置iptables

`iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o venet0 -j MASQUERADE`
(我的openvz出口网卡是venet0，须根据自己的机器相应修改)

`iptables -I FORWARD -s 192.168.10.0/24 -j ACCEPT`

`iptables -I FORWARD -d 192.168.10.0/24 -j ACCEPT`

`service iptables save`

`service iptables restart`

`service pptpd start`



参考

https://gist.github.com/leadfast/8213224

http://www.sulabs.net/?p=275

http://www.centoscn.com/image-text/install/2014/0125/2402.html
