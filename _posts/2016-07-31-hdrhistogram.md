---
layout: post
title: "谈谈 hdrhistogram 和 metrics"
description: ""
category: "learning"
tags: [java, hdrhistogram, stat, metrics]
published: true
---

最近在做rcp框架的 metric 收集部分，说到metric 收集，一般第一个想到工具便是 dropwizard 的 metrics。

然后，看了Hytrix 的代码后发现，他用了 [hdrhistogram](https://github.com/HdrHistogram/HdrHistogram) 来统计latency。

后面将说说 这2个东东。

## hdrhistogram

### 作者

首先，关于 hdrhistogram 作者 Gil Tene 是 Azul System 的CTO，说 Azul 可能很多人不知道，不过如果说，RednaxelaFx 很多人都知道，具体介绍可以看[这里](http://www.zhihu.com/question/24938498)

### 原理

首先 histogram 就是一个直方图

而一般来说我们要实现一个直方图可以用数组表示，数组的每个值表示 一个区间的大小。

而分割区间一般有2种方式

1. 线性的，也就是对于 0..R 如果用 B 分隔， 那么总共可以分隔 R/B 个格子，数组的size也是 R/B
2. 指数的，也就是基于指数分隔，好处是可以减少空间。

举个例子，比如 10w 个数字

采用 1 的分隔方法需要 1000 个buckets, 每个bucket 可以表示100个值:  `[1..100][101..200]...[99,901..100,000]`

采用 2 的方式 只需要 17 个 buckets， `[2^K..(2^(K+1)-1)]` e.g. `[1..1][2..3][4..7]...[65,536..131,071]`

当然以上都有个缺点，需要一个可预估的最大值。超过最大值的数字将存不进来。

那么 hdrhistogram 是如何解决上面的问题的呢？

    对于分隔buckets，hdrhistogram 同时采用了 1 和 2，他内部有2个 index, bucketsIdx 和 subBucketIndex. 前者是指数的 后者是线性的，也就是说可以根据业务精度需求控制后面的线性的大小。

这也是为什么初始化时需要填写2个参数

```java
Histogram histogram2 = new Histogram(3600000l, 1);
```

他根据最大值，得出需要的指数大小，再根据自己配置的线性值

$$ = 10 * log_{10} (x / y) $$

根据代码，baseBucket count 约为 $$ 2^(baseCount+5) $$ 至于为什么要＋5， 因为最小格的大小时从32 开始计算的 (太小的话，反倒是浪费空间)

而 subBucket 的大小为 根据 文档建议的 0-5 大小分别为

```
2
32
256
2048
32768
262144
```

一般定义在2，3左右，定义的越大，约占空间，对了，可以用 `histogram.getEstimatedFootprintInBytes()` 打出占用的空间。

而对于可预估的最大值，提供了一个 autoResize 的功能，不过这个resize 和 arrayList 一样，有 arrayCopy 的成本，所以如果业务上能知道最大值，尽量不要使用autoResize，ps, 官方的autoresize 是从 min 1,  max 2 开始的，太尼玛坑爹了。

### 总结

hdrhistogram 底层其实就是个数组，所以保证了极快的写入速度，也因为如此，所以获取metric的话需要遍历数组统计，所以他非常适合 写多读少的 metric 业务，ps 我们用它来记录rpc 的 latency。


## dropwizard's metric

dropwizard 的 metric 也同样提供了 histogram 的功能.

他提供了 4 种收集方式

1. ExponentiallyDecayingReservoir 指数级别的
2. UniformReservoir   均匀采样
3. SlidingWindowReservoir   类似ringbuffer 的存最近n条记录
4. SlidingTimeWindowReservoir   在3的基础上，增加了时间窗口

根据个人看法，1 的实现更适合我们的业务，而且和 hdrhistogram 的实现更类似。

不过 根据源码来看，他比 hdrhistogram 多了2个开销，

1. 他用 ConcurrentSkipListMap 来做 hdrhistogram 的bucket 干的事情，虽然 skipList 很快，但肯定快不过数组。
2. 他没有 可预估最大值的概念，所以每当插入 比当前空间最大值还大的值时，它需要好多事。。请看一下 code snippet

```java
if (newCount <= size) {
    values.put(priority, sample);
} else {
    Double first = values.firstKey();
    if (first < priority && values.putIfAbsent(priority, sample) == null) {
        // ensure we always remove an item
        while (values.remove(first) == null) {
            first = values.firstKey();
        }
    }
}
```


# 总结

如果你的业务，需要metric的数据无法预估，那么dropwizard 的 histogram 更适合你。

不过对于 可预估最大值的，并且希望写入代价最小的业务来说，还是 hdrhistogram 更合适。


# reference

http://psy-lob-saw.blogspot.jp/2015/02/hdrhistogram-better-latency-capture.html


