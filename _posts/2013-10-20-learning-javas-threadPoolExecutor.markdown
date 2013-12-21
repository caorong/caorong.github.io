---
layout: post
title: "java原生线程池-threadPoolExecutor"
tagline: " "
description: "-desc-"
category: java
tags: [java, threadpool]
---
{% include JB/setup %}


最近再看[webmagic](https://github.com/code4craft/webmagic)的源码，发现他的底层线程池也用了java collections自带的ThreadPoolExecutor。之前一直是抱着拿来主义的态度，但其实还是有有必要仔细研究一番的。

ThreadPoolExecutor 是一个可定制性非常强的线程池。
{% highlight java %}  
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
{% endhighlight %}

从参数可看出，可以定制
######poolsize, 持久存在poolsize

######maxPoolSize  最大可存在poolsize

######keepAliveTime,unit(这2个是闲置线程延迟销毁的时间，及时间单位)

######BlockingQueue 线程queue，我目前用得java6目前提供的6中实现

	ArrayBlockingQueue
	一个固定大小的
	[阻塞队列](http://ifeve.com/blocking-queues/)，放满后不可再放入,可以定义取数据是否按照fifo的顺序还是随机顺序,
	超过定义的size后会throw java.lang.IllegalStateException 的异常。


	DelayedWorkQueue	
	这个queue是一个静态方法，是ScheduledThreadPoolExecutor（ThreadPoolExecutor的扩展，可让线程对象延迟执行）的内部静态类，
	专用queue。不再本篇的讨论范围之内。

	DelayQueue	
	这是无界阻塞队列，队列中的对象需要继承Delayed接口，满足延迟条件才可以使用。

	LinkedBlockingQueue	
	顾名思义，用链表实现的阻塞队列，效率比较高。最大大小为Ineger.MAXVALUE

	PriorityBlockingQueue	
	同样也是个无界阻塞队列对象需要继承Comparable接口，使对象可以进行优先级比较。

	SynchronousQueue	
	一个没有容量的队列，每个插入操作必须等待另一个线程的对应移除操作。就像一个通道一样。
	webmagic的ThreadPool的queue就用了此queue

######ThreadFactory 执行程序创建新线程时使用的工厂。（可选）
Executors也提供了默认的ThreadFactory方法：Executors.defaultThreadFactory()
当然也可以使用apache common lang3的BasicThreadFactory他在Executors.defaultThreadFactory的基础上做了层简单的封装

######handler 超出线程范围和队列容量而使执行被阻塞时所使用的处理程序（可选）
ThreadPoolExecutor已经默认提供了4种handler
分别是
ThreadPoolExecutor.CallerRunsPolicy()  将溢出task交给执行此ThreadPoolExecutor线程执行，一般是主线程。
ThreadPoolExecutor.AbortPolicy()  顾名思义，将任务抛弃,会抛出一个RejectedExecutionException 错误。
ThreadPoolExecutor.DiscardOldestPolicy()  抛弃旧任务
ThreadPoolExecutor.DiscardPolicy() 同AbortPolicy，但不会抛错


webmagic创建的线程池的代码
{% highlight java %}  
public static ExecutorService newFixedThreadPool(int threadSize) {
		if (threadSize <= 1) {
			throw new IllegalArgumentException("ThreadSize must be greater than 1!");
		}
		return new ThreadPoolExecutor(threadSize - 1, threadSize - 1, 0L, TimeUnit.MILLISECONDS,
				new SynchronousQueue<Runnable>(), new ThreadPoolExecutor.CallerRunsPolicy());
	}
{% endhighlight %}
使用作为通道的SynchronousQueue，当所需线程超过规定的线程后，将超过线程池size的当前任务交给主线程执行。（CallerRunsPolicy）
因为webmagic的主循环为一个Thread，里面起一个ThreadPool，所以超出size的task将有Thread执行，不会影响main函数，（其实main也就起了个Thread）
 


