---
layout: post
title: "ssh 的正确使用方法"
description: ""
category: "life"
tags: [aria2c, ssh, scp]
published: true
---

做了快3年多开发，结合最近的情况，总结下，一些可以提升生产力的正确使用姿势。

## private key

很多时候 我们都是通过 `ssh root@xx.xx.xx.xx` 登陆服务器，然后手输密码。

但其实不论从方便性或者从安全性来说，用公私钥登录服务器更好。

ubuntu 的话 自带 `ssh-copy-id` 的工具可以帮助在远程 server 里添上自己的公钥，只要在这次输一次密码，以后都不用输密码。

mac的话 默认没有，可以自己弄一个简化版的

```shell
~ ᐅ cat /usr/local/bin/ssh-copy-id
#!/bin/bash

cat ~/.ssh/id_rsa.pub | ssh $1 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## ssh configfile

如果每次登录都要输入ip，那么如果你要登录的服务器超过100的话很难想象如何记住这么多ip.

但其实可以通过 ssh config file 帮助给你的服务器添加别名

man的官方解释

```nohighlight
-F configfile
       Specifies an alternative per-user configuration file.  If a configuration file is given on the command line, the system-
       wide configuration file (/etc/ssh/ssh_config) will be ignored.  The default for the per-user configuration file is
       ~/.ssh/config.
```

根据man 可以知道 *nux 在 /etc/ssh/ssh_config 有一份默认配置, cp 到 自己的 ~/.ssh/config 下作为自己的配置

然后 添加方式如下

```nohighlight
Host vps5
HostName 1xx.xx.xx.xx
Port 22
IdentityFile ~/.ssh/id_rsa
User root
```

然后可以通过 `ssh vps5` 登录服务器


## proxy jump hosts

一般公司的线上环境都是一个独立的内网环境，需要先登录 一台跳转机，然后跳转到内部机器。

我们自然也是可以 一键 直接进入内部机器。


下面是将vps3 作为gateway，访问vps5。

注： 需要在vps3 上 可以用默认privatekey 登录vps5

```nohighlight
ssh -o ProxyCommand="ssh -W %h:%p vps3" vps5
```

或者 

```nohighlight
ssh -tt vps3 ssh -tt vps5
```

Openssh 不同的版本支持可能不同，具体可以参考 下面referer 的wiki

这个不仅仅用于跳转机。我自己的vps3 在日本，vps5在美国，通过vps3 做代理可以让我访问更顺畅。

## proxy transport file

既然可以代理登录，那就可以代理传文件了。

scp 是一个基于ssh 的 remote copy program (from man)

所以 可以沿用 ssh config `~/.ssh/id_rsa`

由于文件比较大，所以需要选择更合适的 加密方式，以及压缩，以便提高效率，举个例子


```nohighlight
Host proxyed_vps5
HostName vps5IP
# setisify gateway private key position for connect remote %h %p
ProxyCommand ssh -C -c blowfish -l root -i ~/.ssh/vps3_id_rsa vps3IP -W %h:%p
# this is private key for destination on local machine
IdentityFile ~/.ssh/vps5_id_rsa
Compression yes
Cipher blowfish
Port 28030
User root
```

当然也可以先配置一个 `vps3` 再配置一个 `vps5` 然后 ProxyCommand 里面就不用 -c -C -i 这种了

这里我配置了一个 vps5的代理，将vps3 作为 网关。

在 `ProxyCommand` 里面 配置gateway 的选项: 表示通过 root 和 本地 秘钥 通过 `-C` 启用压缩 `-c blowfish` [blowfish 加密效率更高](http://www.hypexr.org/linux_scp_help.php) (ps: 压缩挺费cpu的)

在 ssh_options 里面 用 Compression Cipher 代替。

`-W` 官方解释 

     stdio forwarding (-W) mode to "bounce" the connection through an intermediate host.

`%h` `%p` 分别是 `HostName` 和 `Port`

上面的2个 IdentityFile 都在本机。 

看一下效果:


```nohighlight
~/Documents/xxx ᐅ scp  -r proxyed_vps5:download/xxx-319 ./
xxx-319.mp4                                                                                                    1%   38MB   2.3MB/s   21:52 ETA^CKilled by signal 2.
Killed by signal 2.
~/Documents/xxx ᐅ scp  -r vps5:download/xxx-319 ./
xxx-319.mp4                                                                                                    0% 6768KB 374.0KB/s 2:16:04 ETA^CKilled by signal 2.
```

还有 rsync 同理。

还有，既然能2层，就能 3 层，这是个递归，具体参考 [wiki](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts#Recursively_Chaining_Gateways_Using_stdio_Forwarding) 

# 插曲

之所以会研究下载是因为最近发现，迅雷太恶心了，不但下载速度没以前快，还强制嵌入了一个浏览器。

然后发现国外vps 下载bt 速度非常快，一半1，2G 的 大小，10分钟左右就能下完。于是就找到了这个方法。

ps，如果下载的是 mp4 格式的话，如果前置头 的话，是可以实现scp 边传边播的功能, 毕竟scp 是顺序传输的。。。

# reference

https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts#ProxyCommand_with_Netcat

http://www.hypexr.org/linux_scp_help.php

