---
layout: post
title: "禁网以来"
description: ""
category: "interest"
tags: [bitotrrent,magnet,python,lifan]
---

话说自从禁网以来，不能继续无脑去百度网盘离线下载了，遂不得不补一些bt相关的原理，满足某些需要＝ ＝

[bt协议](http://zh.wikipedia.org/wiki/BitTorrent_(%E5%8D%8F%E8%AE%AE) ，传说中下载的人越多，下载速度就越快的协议。他的原理是依靠tracker server 来做中间人，将下载者们互联起来。

而tracker server 的地址是存在torrent文件里的，那么由于一些老的种子的自带的tracker server已经挂了，或者被墙了。于是给种子加tracker server信息势在必行

tracker server数据可以网上在线加，还有些客户端比如Transmission，uTorrent都可以加，Transmission只能一个一个加，uTorrent只能给一个种子批量加。都比较蛋疼。推荐用脚本批量加。[具体可以参考此处](https://github.com/caorong/Bencode-Torrent-Editor)

话说上周意外找到了个[福利](https://github.com/youxiachai/lifandb)。

于是赶紧下了个uTorrent，下载之，发现竟然无法下载，竟然不能下载。

直接磁力链不能下，那么我是否可以将magnet转换成种子呢？


于是github上找啊找搜到了[这个](https://github.com/danfolkes/Magnet2Torrent),顾名思义，磁力链转种子。很简单的代码，可是使用后发现怎么卡着不懂，看了源码，然后调试后发现，原来卡在了这里。

```python
print("Downloading Metadata (this may take a while)")
while (not handle.has_metadata()):
    try:
        sleep(1)
    except KeyboardInterrupt:
        print("Aborting...")
        ses.pause()
        print("Cleanup dir " + tempdir)
        shutil.rmtree(tempdir)
        sys.exit(0)
ses.pause()
print("Done")
```

while死循环着去拿metadata信息，然后我去吃饭，吃饭完，玩好游戏发现，还在那里跑。。。

（话说我猜测，uTorrent不能下和此信息拿不到有关系，一定是缺了什么必须的参数）


需要补一补[磁力链接](http://zh.wikipedia.org/wiki/%E7%A3%81%E5%8A%9B%E9%93%BE%E6%8E%A5)是啥。

磁力链就是利用了所谓的DHT技术，可以无在Tracker的环境下下载，感觉有点像没有中心server的ed2k

按照wiki上说的，应该更快的找到下载的人才对，可结果确实怎么都搜不到再加上看不懂c＋＋代码。。。于是还是想别的办法将magnet换成bt torrent吧。

于是网上搜到了这3个仓库

- torrage.com
- torcache.net
- zoink.it

暂时只能退而求其次用这个下bt torrent了。话说网上竟然由这种仓库，是不是说明大家还是觉得直接用DHT找队友目前还是很不靠谱啊！！！


