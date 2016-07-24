---
layout: post
title: "pure async"
description: ""
category: "learning"
tags: [java, async, future]
published: true
---

谈谈 java 的异步

最近在搞公司的rpc 框架，花了1 多月的时间，参考 dubbo 和 motan 弄。

最近快弄完了，准备用client 的接入层，这时想到个问题，是不是 我提供了 future 的接口，就是异步了？

比如[知乎](https://www.zhihu.com/question/31440152)上的例子:


```java

fooService.findFoo(fooId); 
Future<Foo> fooFuture = RpcContext.getContext().getFuture(); 
 
barService.findBar(barId); 
Future<Bar> barFuture = RpcContext.getContext().getFuture(); 

Foo foo = fooFuture.get(); // 如果foo已返回，直接拿到返回值，否则线程wait住，等待foo返回后，线程会被notify唤醒。
Bar bar = barFuture.get(); // 同理等待bar返回。


```

这仅保证了我们的应用花费的时间 由原本同步的 ttl(foo) + ttl(bar) 变成了 max(ttl(foo), ttl(bar))

运行这段代码的线程依旧阻塞，当又有请求来的话，又需要申请新的线程处理。。。

理想的异步是什么？应该是类似 脚本语言(node,python) 的event driven 的方式。

而所谓的 event driven 就是 原本程序需要 blocking 住等待返回，现在不blocking，将回调放到内存，等出结果后让系统通知你，然后你在调回调。

而在脚本语言 比如node `让系统通知` 是依赖 libev 封装的 epoll/kqueue.. 实现的，并且只适合io 操作。

然而在java 里面，我们这里虽然可能是 io 操作，但很多情况下，并无法直接利用 类似libev 的东西，(比如jdbc 模型是同步的, 不过 twitter 的 [finagle](https://github.com/twitter/finagle) 将大多数常用的都实现了异步)，

也就是说，类似node 并发量大的话，仅仅多耗费些内存(存 fd callback mapping)。

而 java 的话，不仅要耗费上面的内存，还要自己实现 `通知` 的机制。

这里就我所知，guava 的 `ListenableFuture` 实现了这样的功能。

```java

AsyncFunction<String, Integer> barFuture = new AsyncFunction<String, Integer>() {
      @Override
      public ListenableFuture<Integer> apply(Integer barId) {
        barService.findBar(barId);
        return RpcContext.getContext().getFuture();
      }
    };

ListenableFuture<Integer> resultFuture = Futures.transform(fooFuture, barFuture);

    Futures.addCallback(resultFuture, new FutureCallback<Integer>() {
      @Override
      public void onSuccess(Integer result) {
        System.out.println(result);
        // ....
      }

      @Override
      public void onFailure(Throwable t) {
        t.printStackTrace();
      }
    });
```

看一下她的实现，分析下开销。

SettableFuture 底层内部其实是一个aqs，当对 future 做 `get` 操作的时候，就会park住当前线程。

但如果用 她的transform 的方式，构造一条链式的future(它内部实现链式的通知)，内部没有get。

没有一次调用，我的开销就是 一个SettableFuture 对象，一个runnable 对象(通知), 实现是一个FakeExecutor，并没有 创建 Thread 的开销。 

就结果来说也就实现了 靠内存 提高了并发。


不过以上都针对 java7以内，java8 新增了个 `CompletableFuture` 他收集了所有ListenableFuture in Guava 和 SettableFuture的特征，

再加上lambda 的链式调用，写起来比上面简单多了。。。

java8 的例子

```java

final CompletableFuture completableFuture1 = new CompletableFuture<String>();
final CompletableFuture completableFuture2 = new CompletableFuture<String>();

completableFuture1.thenAcceptBoth(completableFuture2, (a, b) -> {
      System.out.println(a);
      System.out.println(b);
    });
```

不过java8 的get 没有再使用 aqs ，而是依赖 `Signaller`, 不过设计思路都是 将return
task 放进 一个queue 里面，blocking操作用 LockSupport.park 实现。


# reference

https://www.zhihu.com/question/31440152

http://ifeve.com/introduce-abstractqueuedsynchronizer/

https://leanpub.com/whatsnewinjava8/read

http://www.importnew.com/10815.html

