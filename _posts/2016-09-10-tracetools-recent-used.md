---
layout: post
title: "最近挖坑用到的一些性能调试工具"
description: ""
category: "work"
tags: [java, aspectj, histogram, jmc]
published: true
---

话说之前一段时间，一只在找为啥用了netty后性能反倒没有之前好的问题。

然后就想办法找到底是哪里慢了。期间用到了如下几个工具

## java 火焰图

之前在网上看到 别人 用 [火焰图分析java的文章](http://colobu.com/2016/08/10/Java-Flame-Graphs/), 于是就想拿来试试看。

不过翻译的文章使用的还是 Brendan Gregg 使用的老方法，新方法(更方便)在 [netflix 的 blog] (http://techblog.netflix.com/2015/07/java-in-flames.html)上.

跟着官方教程来，中间没遇到什么坑，只不过得到的结果是 cpu 大部分用在了 encode decode 的 string.getByte上，当时就想着很正常。从这个上看不出什么。。

火焰图的作用是用来查看cpu 热点在哪儿。当时我一直想是不是我在某个潜在的地方踩了坑，在哪里阻塞了。而火焰图在这方面帮不了我。

## aspectj

然后就在想，我想要一个工具，最要能将所有的我自己写的代码的每个方法前后都记录一个 time start, time end, 用来统计每个函数花费了多少时间。

由于之前已经做 metric 时借鉴(copy) 了 netflix 的 hystrix 的 metric 方式，只不过将原来的filter 模型, 就想着直接用aop 改成 切面模型对 所有 自己定义的包名的child包下所有的函数 都 `pointcut` 下。

再将 `thisJoinPoint.getSignature` 作为key，各自分配一个 Histogram，纪录每次的执行花费时间。

代码很简单,实现在[这里](https://github.com/caorong/aj-benchmark/blob/master/src/main/java/org/cr/aspect/TimeAspect.aj)

这次得出了, 比较接近的答案，发现慢在自定义的 decoder encoder 上面。但是具体时慢在第几行，还是不知道。

于是就考虑着是不是再把netty 的包也 pointcut 下，但是如果这样的话，

1. 人肉分析肯定是不行的了。得弄个类似火焰图的可视化才行，尼玛 又是个大工程了。
2. 不知道这么多函数内存够不够。。

这时 意外的再微博上 发现了[这个](http://vdisk.weibo.com/s/C9LV9iVqfbYUU), 并在地铁上看的时候意外发现好几个案例都用了叫 jmc 的工具发现内存问题。

## jmc

jmc 全称 [Java Mission Control](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwjei921q4XPAhUBwI8KHdlUA6gQFggeMAA&url=http%3A%2F%2Fwww.oracle.com%2Ftechnetwork%2Fjava%2Fjavaseproducts%2Fmission-control%2Fjava-mission-control-1998576.html&usg=AFQjCNGPqiniTtNK81k-67yazyq4XEm_Cw&sig2=oDZoU1QRY4SJ227WVN9TaQ)

虽然很多人都用这个查到了内存问题啥的，但是他的功能远远不止内存。

简单说下他的强大的功能

1. 内存上，可以统计出在ThreadLocal / 非ThreadLocal 中新申请的 `哪个` 对象 占了总申请对象的百分比，以及 对象 是在哪个函数被申请的，以及改函数的调用连。。
2. cpu上，和内存类似，可以找出是什么包 是热点，并对堆栈进行追踪
3. Thread上，同上
4. IO， 系统， 事件 我都没用到。

然后，瞬间发现了问题所在, 并且是个超级低级的错误

```java
LOG.debug("decode bytebuf is:\n{}", ByteBufUtil.prettyHexDump(buf));
```

卧槽，每次encode，decode都执行了 ByteBufUtil.prettyHexDump，fuck！！

总结 jmc 神器，在jdk7u40 后才有，java8 之后的可以少敲一个 入参，还有最好的文档在 jmc 自带的文档

配置远程调试最简单的方式就是把所有安全相关的都关掉 = = 

```
java -Dcom.sun.management.jmxremote.port=7091
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false MyApp
```

## 随想

话说既然有了 jmc，并且我猜测 netflix 献上已经开始 java8 了，为啥弄哪个火焰图呢？

我在使用了火焰图后事后发现，其实火焰图也能看出上面的问题，但是太隐蔽了，因为从java层到虚拟机层 再到 system call 调用链太深了，而火焰图是根据 高度来找cpu 热点的。而热点都细的的和什么一样。。。肉眼好难分辨 谁更高一些 ＝ ＝

不过，他有一个好处，也是最最重要的是对性能基本无伤，可以在线上直接使用。

根据 netflix 自己说的，java 8 提供的 这个 选项 `-XX:+PreserveFramePointer` 仅仅损伤 `0% - 3%`, 线上完全可以放心大胆的开着。然后系统有问题了，直接画个火焰图看看是哪里的问题。


## flame in python

话说，今天去了 pycon，收获了 eleme 的 [大神的大开脑洞的分享](https://github.com/wooparadog) 他们是怎么做profiling 的

我们公司，java6 的古老体系，用的 aspectj 切每个关键函数的 time cost

这么做确实可以做到对业务方无侵入，但是其实还是要织入很多代码的，这些代码还是会或多或少占用些资源

而他们直接用 systemtab 用来 probe c 的方法 probe python，不用侵入业务方资源，业务方也不用事先准备什么，直接可以 profile。

虽然它们用这个解决的问题是因为 他们用的 gevent 无法直接用 cprofile 找瓶颈。

不过，他的埋点是依赖了 redhat 或者 centos 上面的自带的python 会帮你埋点，(不然需要自己手动改python源码埋点，或者[用python_debug跑](https://github.com/emfree/systemtap-python-tools) 不过这个方式不可能在线上跑)

下面基本摘自他的ppt

redhat/centos/fedora 的埋点是在每个 python 函数的 entry 以及 return 得地方纪录了3个参数

```
python.function.entry filename:string functname:string lineno:long
python.function.return filename:string functname:string lineno:long
```

依赖这个，于是自己写 stp 可以用到


```stp
global pystack

probe ves.function.entry = 
process.library(@1).mark("function__entry")
{
    filename = user_string($arg1);
    funcname = user_string($arg2);
    lineno = $arg3;
}

probe ves.function.return =
process.library(@1).mark("function__return")
{
    filename = user_string($arg1);
    funcname = user_string($arg2);
    lineno = $arg3;
}

probe greenlet = process.library(@2).function("g_switch@greenlet.c")
{
    pystack = $target;
}

probe ves.function.entry
{
    printf("%d => %s => %s in %s:%d\n", pystack, gevent_indent(1), funcname, filename, lineno);
}

probe ves.function.return
{
    printf("%d => %s => %s in %s:%d\n", pystack, gevent_indent(-1), funcname, filename, lineno);
}

probe greenlet {
}
```


```
stap -x PID gevent.stp "/lib64/libpython2.7.so.1.0" "/lib64/python2.7/site-packages/greenlet.so"
```

然后就能看到 

```
120133424 => 1470406350825948 gunicorn: worke(1804):  <= get in /lib64/python2.7/_abcoll.py:365

120133424 => 1470406350925948 gunicorn: worke(1804):  <= get in /lib64/python2.7/_abcoll.py:360
```

类似这样的log

ps 为什么可以这么弄，因为 cPython 其实就是 c, 这里的 cPython 指非pypy 的官方pyhton

## 后记

话说，然后我将他的代码在我公司的测试环境(centos6.5) 试了下，发现不行，貌似我们的 centos 并没有给python事先埋点 orz，应该并非所有的redhat系都埋点了。 orzzzz

```
[root@test132 python-systemtap]# stap -x 45742 gevent2.stp "/usr/lib64/libpython2.6.so.1.0" "/usr/lib64/python2.6/site-packages/greenlet.so"
semantic error: while resolving probe point: identifier 'process' at gevent2.stp:3:7
        source: probe process.library(@1).mark("function__entry")
```


# reference

http://techblog.netflix.com/2015/07/java-in-flames.html

http://www.ibm.com/developerworks/cn/java/j-lo-performance-analysissy-tools3/index.html

https://github.com/emfree/systemtap-python-tools

https://github.com/wooparadog/python-systemtap