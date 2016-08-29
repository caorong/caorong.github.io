---
layout: post
title: "netty 踩坑记"
description: ""
category: "programming"
tags: [java, netty4, ByteBuf]
published: true
---

以下netty 版本为 4.1.x

最近项目的收官阶段，开始压测，然后发现效果没有想象的好，再用 `gcutil` 看内存，发现竟然有严重的内存泄露！

仔细研究了一下，发现是因为我的使用方式不对。

最初开发的时候，没有研究过netty的底层实现，但是看过 [twitter发的对netty4 的 ByteBufPool 的测评](https://blog.twitter.com/2013/netty-4-at-twitter-reduced-gc-overhead)，
于是就在写代码的时候就想着尽量池化。

错误的方式 差不多是下面这样的代码.

```java
// decode 部分, 重用 bootstrap 用 PooledByteBufAllocator 分配的Bytebuf, 交给handler
@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
  int available = in.readableBytes();
  if (available > 4) { // read frame handle half package
    in.markReaderIndex();
    final byte[] i32buf = new byte[4];
    in.readBytes(i32buf);
    // 这个total 不包括frame 的4 个字节
    int totalSize = decodeFrameSize(i32buf);
    if (available < totalSize + 4) { // read frame handle half package
      in.resetReaderIndex();
    } else {
      Constants.MessageType messageType = isXXProtocol(in);
      Message message = null;

      switch (messageType) {
        case XX1:
          message = new Message();
          message.setBody(ink.readRetainedSlice(totalSize));
          out.add(message);
          break;
        case xx2:
          break;
      }
    }
  }
}

// handler 部分，接受 decode 出来的 Object

@Override
protected void channelRead0(final ChannelHandlerContext ctx, final Message request) throws Exception {
  threadPoolExecutor.execute(new Runnable() {
    try {
      Message response = messageHandler.handle(request); // 这里的 request 就是decode的msg，里面包含了IO线程分配的Bytebuf 
      if (ctx.channel().isActive()) {
        ctx.writeAndFlush(response, ctx.voidPromise());
      }
    } catch (Exception e) {
    } finally {
      assert request.getBody().refCnt() == 1;
      ReferenceCountUtil.release(request.getBody()); // 释放上面的bytebuf
    }
  }
}
```

看起来代码没有问题，但是事实上由于netty4 的cache特性，造成了内存泄露。

netty4 有个 [Thread-local object pool](http://netty.io/wiki/using-as-a-generic-library.html#thread-local-object-pool) 

netty 在很多地方都用它来做cache

用来做 cache。做什么cache呢，
用来做ByteBufPool 的cache，也就是说每当有请求server 的时候，先从Threadlocal 找之前cache 过的Bytebuf，

当使用 worker 的optional 是 PooledByteBufAllocator 时 netty 处理请求时 的内存分配流程是这样的

```text
                          +-----------------+  +-----------------+
                          |IO thread for write |IO thread for read    1. Inbound 字节流存入ByteBuf中。
   3. 将encode好的Bytebuf  |                 |  |                 |    Bytebuf 优先从Threadlocal获取，
   写出到socket，回收Bytebuf|            ^    |  |   +             |    ThreadLocal没有，则再从pool中获取。
   到 ThreadLocal         |            |    |  |   |             |
                          |            |    |  |   |             |
                          |            |    |  |   |             |
                          |            |    |  |   |             |
                          |            |    |  |   |             |
                          |            |    |  |   |             |
                          |            +    |  |   v             |
                          |                 |  |                 |
                          |       +----------------------+       |
+-------------------+     |       |         |  |         |       |
|  heap or          | +---------> |   thread local cache |       |
|  direct           |     |       |         |  |         |       |
|                   |     +-------+---------+--+---+-----+-------+     2.一般来说在business handler 中将解码的数据
|  ByteBuf pool     |     |                        |             |     放到自己的业务线程池，防止由于阻塞降低IO thread
|                   | <----------------------------+             |     的吞吐
|                   |     |           +------------------+       |    +------------------------+
+-------------------+     |           |                  |       |    |  business thread       |
                          |           |decoder           |       |    | +--------------------+ |
                          |           |                  |       |    | |                    | |
                          |           |                  |       |    | +--------------------+ |
                          |           |business handler+------------> |                        |
                          |           |                  |       |    | +--------------------+ |
                          |           |                  |       |    | |                    | |
                          |           |encoder           |       |    | +--------------------+ |
                          |           |                  |       |    |                        |
                          |           |                  |       |    |      .......           |
                          |           +------------------+       |    |                        |
                          |  pipeline is also in IO thread       |    |                        |
                          +--------------------------------------+    +------------------------+

```

所以，如果我将 1 中 获得的Bytebuf 在2 中的业务线程释放，于是 `ByteBuf.release` 的流程将会把Bytebuf 存入到当前的Threadlocal。(这个Threadlocal 无法释放) 

于是内存泄漏了。

那么问题来了，既然netty 已经支持了 BytebufPool 为何还要用 Threadlocal 做Cache呢？

这里有个官方的 [benchmark](https://github.com/netty/netty/pull/2284) 解释了主要是因为 directByteBuf 的获取比较慢。。也就是说，这个优化主要是针对 DirectBytebufPool的

总结来说，如果要使用 PooledByteBuf，一定要注意 allocate 和 release 是同一个线程

但是这里要注意，我们的IOThread 可以配置 `>1` 的, 因为有时我们希望我们的 encoder 和 decoder 能过利用多个核, 这里要注意，在你使用多个线程的时候，你的 ThreadLocal 也变成了多个，
这些Threadlocal 都会进入老年代，并且永远不会被释放，需要压测看看，配置合理的内存大小，避免由于内存过小，造成频繁full gc

对了，这个cache的特性还能够关闭 具体代码在 `io.netty.util.Recycler` static 代码块, 可以通过 `System.setProperty("io.netty.recycler.maxCapacity", "0");` 或以 
java param 的方式 `-Dio.netty.recycler.maxCapacity=0` 关闭

还有PooledByteBuf 的cahce是这样子的cache，他cache的是自己的是自己的对象，所以，这个cache 也继承了整个netty 的类似jmelloc 的内存分页方式, 并且他的大小是和Pooled Bytebuf 大小是一样的。

```java

// PooledArena
@Override
protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
    if (HAS_UNSAFE) {
        return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
    } else {
        return PooledDirectByteBuf.newInstance(maxCapacity);
    }
}

// PooledUnsafeDirectByteBuf
static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
    PooledUnsafeDirectByteBuf buf = RECYCLER.get();
    buf.setRefCnt(1);
    buf.maxCapacity(maxCapacity);
    return buf;
}
// PooledHeapByteBuf 
static PooledHeapByteBuf newInstance(int maxCapacity) {
    PooledHeapByteBuf buf = RECYCLER.get();
    buf.setRefCnt(1);
    buf.maxCapacity(maxCapacity);
    return buf;
}
```

所以如果你用的是 PooledHeapByteBuf 如果有 2个 IOThread 那么 cache 的总数是3。

所以，如果关心内存的使用情况， 了解netty内存池的分配非常重要，关于这个[发现这篇文章写的很好](http://blog.csdn.net/pentiumchen/article/details/45372625)

然后根据自己的业务情况，以及可能的线程数，预估 最大的内存耗费. 或者直接压测， 23333

还有，如果 业务线程也使用了PooledBytebuf 的话， 同理.


于是，知道了这个原因后, 为了验证下，于是在 decoder 里面用 UnpooledHeapBytebuf 做一次内存拷贝，然后试试看是否能修复。

代码如下 

```java
switch (messageType) {
  case xx1:
    message = new Message();
    message.setBody(Unpooled.copiedBuffer(in));
    out.add(message);
    break;
  case xx2:
    break;
}
```

压测后发现，老年代依然会逐步增加，最终导致fullgc

objectDump 后发现大量的Recycler 和 WeakHashMap 对象。明明Unpooled 申请Bytebuf 的时候并没有和ThreadLocal相关，为何还会有Recycle呢？

后来发现是在 business Thread writeAndFlush 的时候, 会区分线程, 如果是在 `IO 线程` writeAndFlush 则会直接调用，不然则是提交一个runnable

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```

提交任务后, 依旧是从 ThreadLocal 中获取 `WriteAndFlushTask` ，这里的 `WriteAndFlushTask` 被重用了。

这里可以肯定，这里的 `WriteAndFlushTask` 一定是会在IO线程释放, 如果和上面的 PooledByteBuf 比较，岂不是内存泄漏了？

我们继续看下去，看释放的时候做了些什么

```java
private static void safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg) {
        try {
            executor.execute(runnable);
        } catch (Throwable cause) {
            try {
                promise.setFailure(cause);
            } finally {
                if (msg != null) {
                    ReferenceCountUtil.release(msg);
                }
            }
        }
    }
```

这个 executor 其实就是 当初 调用 businessThread 的 `ChannelHandlerContext` 这里插播一句， `ChannelHandlerContext` 和 `eventLoop` 是一对多的关系， 前者为一，后者为多

也就是说，这个runnable被 提交到 eventLoop 去了，

```java
// NioEventLoop / SingleThreadEventExecutor#execute
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

ps，这里的逻辑和我以前看tornado 的eventLoop 的 大逻辑差不多，但是 netty 多了不少细节(细节以后有时间再展开，这里就说说大体流程)

通过上面 addTask 将 上面的 `WriteAndFlushTask` 提交到一个taskQueue

```java
// NioEventLoop / SingleThreadEventExecutor#execute
protected void addTask(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (!offerTask(task)) {
            reject(task);
        }
    }

    final boolean offerTask(Runnable task) {
        if (isShutdown()) {
            reject();
        }
        return taskQueue.offer(task);
    }
```

然后就是loop 环节, 

```java

// NioEventLoop#run()

// 仅仅贴部分代码，省去了分配时间比例的代码

protected void run() {
  for (;;) {
    try {
      // tornado 的代码就类似这样，还有 tornado 是先处理挤压的task 再处理IO
      processSelectedKeys();
      runAllTasks();
    } catch (Throwable t) {
      logger.warn("Unexpected exception in the selector loop.", t);
      // Prevent possible consecutive immediate failures that lead to
      // excessive CPU consumption.
      try {
        Thread.sleep(1000);
      } catch (InterruptedException e) {
        // Ignore.
      }
    }
  }
}
```

再loop里面 执行 `WriteAndFlushTask` 

```java
// AbstractChannelHandlerContext.AbstractWriteTask#run()

public final void run() {
    try {
        ChannelOutboundBuffer buffer = ctx.channel().unsafe().outboundBuffer();
        // Check for null as it may be set to null if the channel is closed already
        if (ESTIMATE_TASK_SIZE_ON_SUBMIT && buffer != null) {
            buffer.decrementPendingOutboundBytes(size);
        }
        write(ctx, msg, promise);
    } finally {
        // Set to null so the GC can collect them directly
        ctx = null;
        msg = null;
        promise = null;
        handle.recycle(this); // 还记得前面在BusinessThread 里面 `RECYCLER.get();` 吗， 这里就是回收
    }
}
```

不过看到这里，我知道了大体流程是怎样的了，但是具体的跨线程是如何做到cache的还是没看到(有点跑偏),所以下面着重看Recycler 的get 和 recycle 的实现

```java
// Recyler  @BusinessThread
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
    @Override
    protected Stack<T> initialValue() {
        // 这里的 Thread.currentThread() 就是我们的 Business Thread
        return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacity, maxSharedCapacityFactor);
    }
};

// Recycler#get()  @BusinessThread
public final T get() {
  // 关闭cache 的时候，每次都 new 对象，
    if (maxCapacity == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    // threadLocal 里面放一个 Stack， 
    Stack<T> stack = threadLocal.get(); // 这里的threadLocal 就是上面的 FastThreadLocal
    DefaultHandle<T> handle = stack.pop(); // 通过pop 获得cache的 DefaultHandle
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}

// -------------------------------- different context ----------------------------------------------

// Recycler.DefaultHandle  @EventLoopThread (WorkerThread/IOThread)
private static final FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED =
        new FastThreadLocal<Map<Stack<?>, WeakOrderQueue>>() {
    @Override
    protected Map<Stack<?>, WeakOrderQueue> initialValue() {
        return new WeakHashMap<Stack<?>, WeakOrderQueue>();
    }
};
    
// Recycler.DefaultHandle#recycle()   @EventLoopThread (WorkerThread/IOThread)
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    Thread thread = Thread.currentThread();
    // 这边我们走不到，因为这里是在IOThread里，而stack的thread是business Thread
    if (thread == stack.thread) {
        stack.push(this);
        return;
    }
    // we don't want to have a ref to the queue as the value in our weak map
    // so we null it out; to ensure there are no races with restoring it later
    // we impose a memory ordering here (no-op on x86)
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(stack);
    if (queue == null) {
        queue = WeakOrderQueue.allocate(stack, thread);
        if (queue == null) {
            // drop object
            return;
        }
        delayedRecycled.put(stack, queue);
    }
    // 通过这一步将 DefaultHandle 放回queue
    queue.add(this);
}
```

注意，这里的 `queue.add` 和 `stack.pop` 明明是2个对象，怎么就串联起来了呢

代码比较长，我直接画图表示下

[!wiki/netty/netty.png](/post_images/netty.png)

也就是说 一个Thread 最多cache `256` 个 `WriteAndFlushTask` 对象, 总数又 `IO Thread` 数量决定, 默认最多应该是 IOThreadCount * 256 个对象

超过这个数字的话都直接 new 一次性对象了。

所以这个是不会造成内存泄漏的，所以回到之前的问题，为什么用Unpool还会造成 老年代持续上升。

后来我将 selector 线程 和 IO 线程都改成 1个 然后，然后 print 出每次请求的id(每次请求id 自增)，发现 id 并不同步，降低频率后可以同步，也就是说，

server 没有将buf 写出去，由于是在 businessThread 里写 response 是提交到 IOLoop 里面 的queue 的，然后 IOLoop 里面 是 给 读的IO 事件和别的 所有异步事件 各 50% 的时间 

```java
for(;;){
  // 省略 xxx
  final int ioRatio = this.ioRatio;// 默认是 50
    if (ioRatio == 100) { 
        processSelectedKeys();
        runAllTasks();
    } else {
        final long ioStartTime = System.nanoTime();

        // 先处理IO时间，
        processSelectedKeys();

        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }

    if (isShuttingDown()) {
        closeAll();
        if (confirmShutdown()) {
            break;
        }
    }
  // 省略 xxx
}
```

这个Task 不仅仅是 WriteAndFlushTask 还有 CloseTask， registerTask, 等等返回xxxFuture 的基本都是靠他实现的。

虽然在我们的压测情况下，除了 WriteAndFlushTask, 别的可以忽略不计，还有就是 如果写的太快，造成task 积压过多。

所以，在一个异步的系统里面 设置 [waterMark](http://normanmaurer.me/presentations/2014-facebook-eng-netty/slides.html#11.0) 非常重要

事后，在看netty issue的时候，也有人遇到了同样的问题  https://github.com/netty/netty/issues/5563



# reference

http://blog.csdn.net/pentiumchen/article/details/45372625

http://www.cnblogs.com/rainy-shurun/p/5213086.html

http://redis.io/commands/eval#available-libraries


http://normanmaurer.me/presentations/2014-facebook-eng-netty/slides.html
