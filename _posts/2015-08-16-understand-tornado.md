---
layout: post
title: "Understand Tornado"
description: ""
category: "python"
tags: [python, tornado, yield]
published: true
---

让我们先来写一个hello world

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):

    @tornado.gen.coroutine
    def get(self):
        self.write("Hello, world")

application = tornado.web.Application([
    (r"/", MainHandler),
])

if __name__ == "__main__":
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

上面这个程序表示在 `8888` 端口启动一个webserver, 然后访问后返回 `Hello world`

要了解他的机理，先要理解以下几点

## epoll

为了了解这个，我们得先了解什么是 epoll, 以及 epoll 的编程例子。

这个可以参考参考[这里](http://zoomq.qiniudn.com/ZQScrapBook/ZqFLOSS/data/20100927213110/)的Example 3

## Generator 

python的迭代器 类似ruby 的fiber， 都可以作迭代，异步传值



### 迭代

```python

>>> class Data(object):
...     def __init__(self, *args):
...         self._data = list(args)
...     def __iter__(self):
...         for x in self._data:
...             yield x
...
>>> d = Data(1, 2, 3)
>>> for x in d:
...     print(x)


>>> (i for i in [1,2,3])
<generator object <genexpr> at 0x10fd7ed38>

>>> [i for i in [1,2,3]]
[1, 2, 3]

```


### 传值

举个例子

```python
def foo(n):
    for i in range(n):
        re = yield i
        print "v_%s = %s" % (i, re)

g = foo(2)      # 1
print g.next()  # 2
print g.next()  # 3
print g.next()  # 4

```

上面的代码输出的是：

```
0
v_0 = None
1
v_1 = None
Traceback (most recent call last):
  File "/Users/ryan/Workspace/myself/Tornadjango/app/tmp.py", line 10, in <module>
    print g.next()
StopIteration
```

1 得到一个generator，且在1的时候，foo函数不会有任何的执行。在解释器分析代码的时候，发现foo里面有yield，就已经把它当成一个generator了。

2 得到generator的第一个值 0

3 得到generator的第二个值 1

4 由于generator已经完毕，所以，抛出StopIteration异常

然而yield的返回都是None 


* 注意 generator.next() 等于  next(generator) py2 两种写法都支持，py3 只支持后者

```python 
def coroutine():
    print "coroutine start..."
    result = None
    while True:
        s = yield result
        result = 'result: {}'.format(s)

>>> c = coroutine() # 函数返回协程对象
>>> c.send(None) # 使用 send(None) 或 next() 启动协程  send(None) == next(c)
coroutine start...
>>> c.send("first") # 向协程发送消息,使其恢复执⾏
'result: first'
>>> c.send("second")
'result: second'
>>> c.close() # 关闭协程,使其退出。或⽤c.throw() 使其引发异常
>>> c.send("never recv")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration

```


1. generator.next() 等价于 generator.send(None)
2. 对于一个generator，一定要先调用一个next，或者send(None)。才能再调用send(value)
3. 调用send(value)的时候，会把value的值赋给里面的接收者，且是上一次next的中断位置

关于generator的其他教程

http://blog.bitfoc.us/?p=502

http://dongweiming.github.io/Expert-Python/#22

[stackoverflow 的例子](http://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do-in-python)


## decorator pattern

先讲讲 `@tornado.gen.coroutine` 是什么，干什么用的，这种写法又是什么意思？

其实这是一种装饰模式，java 里面也很常见，比如 thrift client 的配置 

```xml
<bean id="monitorMsgClient" class="com.ximalaya.thrift.client.ThriftClient">
    <property name="clientFactory" ref="monitorCollecterFactory"></property>
</bean>

<bean id="thriftMonitorCollecterClient" class="com.ximalaya.live.president.inf.thrift.client.impl.MonitorMsgThriftClientServiceImpl">
    <property name="monitorMsgClient" ref="monitorMsgClient"></property>
</bean>
```

`com.ximalaya.thrift.client.ThriftClient` 代码

```java
@SuppressWarnings("unchecked")
@Override
public T getObject() throws Exception {
    return (T) Proxy.newProxyInstance(ClassUtils.getCurrentClassLoader(), new Class[]{iface}, new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            ThriftConnection<?> connection = clientFactory.getConnection();
            try {
                T client = (T) connection.getClient();
                return method.invoke(client, args);
            } catch (InvocationTargetException e) {
                throw e.getCause();
            } finally {
                connection.close();
            }
        }
    });
}
```

所以在这里 monitorMsgClient 其实已经不是纯粹的 Iface 对象了，而是一个被修饰过的对象，当然这里这么用的目的是少写代码。

```java

    private Iface monitorMsgClient;

    @Override
    public boolean remindFailMsgOfRadio(String errorCode, String errorMsg, int radioId, int audioSegmentId)
            throws TException {
        try {
            return monitorMsgClient.remindFailMsgOfRadio(errorCode, errorMsg, radioId, audioSegmentId);
        } catch (Exception e) {
            throw new TException(e.getMessage(), e);
        }
    }

```

那么我们回过头再来看 python 的 decorator

```python
>>> def common(func):
...     def _deco(*args, **kwargs):
...         print 'args:', args
...         return func(*args, **kwargs)
...     return _deco
...
>>> @common
... def test(p):
...     print p
...

>>> test
<function _deco at 0x10d996a28>
>>> test(1)
args: (1,)
1

```

这里的 `func(*args, **kwargs)` 相当于 java 的 `method.invoke(client, args);`

同样， 当别人调用 test 函数的时候， 这个test函数已经不是原来的test函数了。

不过tornado 里面用个更高级的 decorator `@functools.wraps`, 至于他的作用，看下面的代码就知道了

```python
from functools import wraps
 
 
def without_wraps(func):
    def __wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return __wrapper
 
def with_wraps(func):
    @wraps(func)
    def __wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return __wrapper
 
 
@without_wraps
def my_func_a():
    """Here is my_func_a doc string text."""
    pass
 
@with_wraps
def my_func_b():
    """Here is my_func_b doc string text."""
    pass
 
 
# Below are the results without using @wraps decorator
print my_func_a.__doc__
>>> None
print my_func_a.__name__
>>> __wrapper
 
# Below are the results with using @wraps decorator
print my_func_b.__doc__
>>> Here is my_func_b doc string text.
print my_func_b.__name__
>>> my_func_b

```

其实就是保证了一些元数据不丢

这里了解了 @decorator 那么跟着看源码 tornado 如何利用 @tornado.gen.coroutine 做到异步



## 猜测tornado的整体流程

通过上面的基础知识，大致可以联想一下tornado的流程

### 最最简易版的 handler 

```python
class MainHandler1(tornado.web.RequestHandler):

    def get(self):
        self.write("123")
```

1. cliet发起请求
2. server 的 epoll 收到个event，请求连接
3. server register client connection 
4. server 的 epoll 收到 register 执行完成，将 request_header 读入内存
5. server 根据 handler + method 调用handler 的相应方法， method 的 默认实现皆为 raise 405
6. method 将output 的string 异步写入 connection

* 其中 4 是不等待3是否完成，3执行完毕后由 epoll 通知4 
* 6 是异步写入connection 同上，回调关闭连接

结束

流程是如此，tornado 在实现的时候，利用了 decorator 和 yield 在实现异步，使用的方式如下

1. 被 decorate 的代码 比如 @gen.coroutine 遇到 yield， 那么我们执行func 得到的必然是 一个generator

2. next(generator) 将带 yield 的func 执行到 yield 处，返回yield 后面的代码块

3. yield 后的代码块执行完成后通过 `通知` 触发 Runner，然后通过 generator 的send 将 yield 右边的结果写入yield左边，实现异步。

注， 上面的 `通知` 在ioloop 里面 有callback 和 epoll.poll 两种实现方式, 后有详细说明。

```python

@functools.wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)  # 1 result 是一个 generator 
        if isinstance(result, types.GeneratorType):
            yielded = next(result) # yielded 为
            #...
            Runner(result, future, yielded)

# ...

class Runner(object):
    def __init__(self, gen, result_future, first_yielded):
        self.gen = gen
        self.result_future = result_future
        self.future = _null_future
        if self.handle_yield(first_yielded):
            self.run()

    def run(self):
    #...
        yielded = self.gen.send(value)
    #...

# 举个简单例子加强理解

def coroutine():
    print "coroutine start..."
    result = lambda x: print('lala %s' % x)
    while True:
        s = yield result
        result = 'result: {}'.format(s)

# >>> c = coroutine()
# >>> c
# <generator object coroutine at 0x10fb9eab0>
# >>> xx = next(c)
# conr start..
# >>> xx
# <function coroutine.<locals>.<lambda> at 0x10fe6d268>  
```

拿到函数后 我自然就可以在`恰当的时机`执行他(比如收到io 完成的通知的时候)


### tips 

在这最简单的情况下我们应该会收到2个 epoll 的 event

不过通过调试发现现实会与想象的不同

1. 在本地请求 比如 `curl localhost:8080/t` 会显示收不到 4 的event，暂时没找到原因，但如果 先ssh 到别的机器在请求就能看到效果...

2. 在满足 1 的情况下，本地用pycharm debug 还是会看不到 4 的event，但是却是是有event的，解决方法，这个event的观察不能 debug下执行，(原因暂未想通) 直接在tornado 源码上增加

```python
# 注意 linux 是 epoll ， 而且是二进制的 看不到代码 ＝ ＝
# .../tornado/platform/kqueue.py  

# 在register 里面打印堆栈

def register(self, fd, events):
    import inspect, traceback
    print('[kqueue register] called - %s %s' % (__file__, inspect.currentframe().f_back.f_lineno))
    for line in traceback.format_stack():
        print(line.strip())
    print()
    ...
```

请在下面地方增加注释，便于发现以上步骤

```python
# ioloop.start()

def start():
    # ...
    with self._callback_lock:
        print('_callbacks', self._callbacks)
        callbacks = self._callbacks
        self._callbacks = []
    # ...

    # ...
    try:
        event_pairs = self._impl.poll(poll_timeout)
        print('event_pairs', event_pairs)
    except Exception as e:

    # ...
```

以上注释便于发现，服务启动后，如果没有请求则会不断打印以下数据，而且debug 某个请求的时候，会停止print 充分说明 ioloop 是单线程的，以及阻塞某个请求会影响整个服务

```
event_pairs dict_items([])
_callbacks []
event_pairs dict_items([])
_callbacks []
event_pairs dict_items([])
_callbacks []
...
```




## @tornado.gen.coroutine

简化版的代码 

```python

def coroutine(func, replace_callback=True):
    return _make_coroutine_wrapper(func, replace_callback=True)


def _make_coroutine_wrapper(func, replace_callback):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        future = TracebackFuture()
        try:
            result = func(*args, **kwargs)  # 如果func 内有 yield 的话，result 是<generator xx>
        except (Return, StopIteration) as e:
            result = getattr(e, 'value', None)
        except Exception:
            future.set_exc_info(sys.exc_info())
            return future
        if isinstance(result, types.GeneratorType):
            yielded = next(result)  #py3 的用法 == result.next()   启动generator， 将func执行到yield 这里，然后返回后面的代码块
            Runner(result, future, yielded)
            return future
            
        future.set_result(result)
        return future
    return wrapper
```


也就是说当我们的被注解的函数

```python
@tornado.gen.coroutine
    def get(self):
        self.write("Hello, world")
```

1. 如果内部不存在 yield 则执行函数后直接返回一个future

被注解的函数内存在 `yield` 的话，那么一开始得到的result是 一个 generator 并没有被执行，而在下面调用next 进行初始化，然后交给下面的 Runner 




## ioloop

然后我们再来看 tornado 的 ioloop 模块

根据上面的 epoll example 我们可以猜到，tornado 的大致流程，一旦有请求过来，先触发一个 epoll 事件。然后做一个 register。 然后

看了代码后，发现她还有callback的

也就是说，ioloop 有2种触发器，一个是 socket 的，一个是 带 timeout 的 callback， socket 是通过epoll 触发代码如下


```python
# ioloop.py

try:
    event_pairs = self._impl.poll(poll_timeout)
except Exception as e: 
    # 收到 socket 连接提醒， 
    # 收到 socket 数据读取完毕提醒
```

一个是 callback, 比如 `@run_on_executor` 他是在 decoretor 里面 将被 装饰的函数 submit 到threadpool 执行

```
# ioloop.py

for callback in callbacks:
    self._run_callback(callback)

def _run_callback(self, callback):
    """Runs a callback with error handling.

    For use in subclasses.
    """
    try:
        ret = callback()
        if ret is not None and is_future(ret):
            # 将依据
            self.add_future(ret, lambda f: f.result())
    except Exception:
        self.handle_callback_exception(callback)
```


## epoll.poll 类型代码解析

```python
try:
    event_pairs = self._impl.poll(poll_timeout)
    print('event_pairs', event_pairs)
except Exception as e:
    ...

self._events.update(event_pairs)
while self._events:
    fd, events = self._events.popitem()
    try:
        fd_obj, handler_func = self._handlers[fd]
        handler_func(fd_obj, events)
    except (OSError, IOError) as e:
        if errno_from_exception(e) == errno.EPIPE:
            # Happens when the client closes the connection
            pass
        else:
            self.handle_callback_exception(self._handlers.get(fd))
    except Exception:
        self.handle_callback_exception(self._handlers.get(fd))
fd_obj = handler_func = None

```

ioloop 通过 收到的 event 调用相应的 handler 




## callback 类型代码解析 (run_on_executor_decorator)

将被注解的 function 提交到 threadpool 执行


```python
# concurrent.py l.365

def run_on_executor_decorator(fn):
    executor = kwargs.get("executor", "executor")
    io_loop = kwargs.get("io_loop", "io_loop")
    @functools.wraps(fn)
    def wrapper(self, *args, **kwargs):
        callback = kwargs.pop("callback", None)
        future = getattr(self, executor).submit(fn, self, *args, **kwargs)
        if callback:
            getattr(self, io_loop).add_future(
                future, lambda future: callback(future.result()))
        return future
    return wrapper
```

提交上去后返回 future 

```python
# gen.py _make_coroutine_wrapper

yielded = next(result)
Runner(result, future, yielded)
``` 

这里的Runner 对象干了很多事情，因为我们将任务交给thread 处理，那么我们怎么知道 task 已经完成

tornado 在初始化 Runner 对象的时候在最后一步先判断 future 是否依旧完成， 如果没有完成则在 ioloop 里面加一个future （add_future）。



```python
# gen.py 

class Runner(object):

    def __init__(self, gen, result_future, first_yielded):
        if self.handle_yield(first_yielded):
            self.run()

    def handle_yield(self, yielded):
        # ...
        if not self.future.done() or self.future is moment:
            self.io_loop.add_future(
                self.future, lambda f: self.run())
            return False

# ioloop

def add_future(self, future, callback):
    """Schedules a callback on the ``IOLoop`` when the given
    `.Future` is finished.

    The callback is invoked with one argument, the
    `.Future`.
    """
    assert is_future(future)
    callback = stack_context.wrap(callback)
    future.add_done_callback(
        lambda future: self.add_callback(callback, future))
```
    
ioloop.add_future 里则给 future 加一个 add_done_callback, 也就是thread task 完成后执行 


```python
lambda future: self.add_callback(callback, future)
``` 

而这个self 时ioloop， 也就是ioloop 里面 轮训的callback 多了一条， 相当于 ioloop 收到了信息。




简化版

```python

def start(self):
    while True:
        try:
            event_pairs = self._impl.poll(poll_timeout)
        except Exception as e:
            # Depending on python version and IOLoop implementation,
            # different exception types may be thrown and there are
            # two ways EINTR might be signaled:
            # * e.errno == errno.EINTR
            # * e.args is like (errno.EINTR, 'Interrupted system call')
            if errno_from_exception(e) == errno.EINTR:
                continue
            else:
                raise

        # Pop one fd at a time from the set of pending fds and run
        # its handler. Since that handler may perform actions on
        # other file descriptors, there may be reentrant calls to
        # this IOLoop that update self._events
        self._events.update(event_pairs)
        while self._events:
            fd, events = self._events.popitem()
            try:
                fd_obj, handler_func = self._handlers[fd]
                handler_func(fd_obj, events)
            except (OSError, IOError) as e:
                if errno_from_exception(e) == errno.EPIPE:
                    # Happens when the client closes the connection
                    pass
                else:
                    self.handle_callback_exception(self._handlers.get(fd))
            except Exception:
                self.handle_callback_exception(self._handlers.get(fd))
        fd_obj = handler_func = None


def add_handler(self, fd, handler, events):
    fd, obj = self.split_fd(fd)
    self._handlers[fd] = (obj, stack_context.wrap(handler))
    self._impl.register(fd, events | self.ERROR)
```

可以看到，ioloop 是一个死循环，他不停的取出epoll 中的事件，然后根据fd，拿出fd 对应的handler, 然后交给handler处理。

结合上面的epoll 例子，我们可以想象，所有的对 IO(socket) 有操作的都会注册到机器环境的 epoll 中，然后循环地获取完成io的事件。

也就是说，利用系统的回调来省去自己对io 的等待。


## 练手

我估计，光看上面的写的也只有我自己能看懂，于是我写一个简化版的带 ioloop， coroutine 以及 run_on_executor 的实现，帮助理解关键部分，理解了后再看上面，并跟着看源码代码应该就能理解了。

```python
#! /usr/bin/env python
# coding=utf-8

from concurrent.futures import ThreadPoolExecutor
from functools import wraps
import time
from threading import Thread
import types


class FakeIOloop(object):
    """
    线程实现的伪ioloop, 真正tornado的是没有这个Thread的
    """

    def __init__(self):
        self._callback = []
        thread = Thread(target=self.start, args=())
        thread.start()
        # thread.join()

    def add_callback(self, cb):
        self._callback.append(cb)

    def start(self):
        while True:
            time.sleep(1)
            # print('cb', self._callback)
            if self._callback:
                x = self._callback.pop(
                    0)  # <function coroutine.<locals>.wrapper.<locals>.<lambda>.<locals>.<lambda> at 0x1051689d8>
                # 该x 就是 coroutine 中的  send_result 函数
                try:
                    x()
                except (StopIteration) as e:
                    # print(e)
                    pass


ioloop = FakeIOloop()


def run_on_executor(*args, **kwargs):
    def run_on_executor_deco(fn):
        executor = kwargs.get("executor", "executor")

        @wraps(fn)
        def wrapper(self, *args, **kwargs):
            future = getattr(self, executor).submit(fn, self, *args, **kwargs)
            return future

        return wrapper

    return run_on_executor_deco


def coroutine(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print('1.', result)
        if isinstance(result, types.GeneratorType):
            # run the self.test
            yielded = next(result)
            print('3.', yielded)

            def send_result(future):
                print('inner send result called !!!!')
                result.send(future.result())

            # 在线程内添加一个完成后的回调，该回调将在ioloop内添加一个callback，并在该callback内执行上面的函数
            yielded.add_done_callback(lambda x: ioloop.add_callback(lambda: send_result(x)))
            return yielded
        else:
            return result

    return wrapper


class TestCls(object):
    executor = ThreadPoolExecutor(10)

    @coroutine
    def getdata_without_sleep(self):
        return "i am not blocked"

    @coroutine
    def getdata_with_sleep(self):
        print('2.', self.test_block)
        result = yield self.test_block('333')
        print('4...', result)
        return result

    @run_on_executor(executor='executor')
    def test_block(self, a):
        time.sleep(2)
        return "finish %s " % a


if __name__ == '__main__':
    tcls = TestCls()
    # 执行内含sleep的func
    tcls.getdata_with_sleep()
    tcls.getdata_without_sleep()


# 1. <generator object getdata_with_sleep at 0x10f713900>
# 2. <bound method TestCls.test_block of <__main__.TestCls object at 0x10f711f28>>
# 3. <Future at 0x10f711f60 state=running>
# 1. i am not blocked
# inner send result called !!!!
# 4... finish 333
```

可见 `getdata_without_sleep` 函数并没有被上个函数的sleep阻塞。 
`getdata_with_sleep` 函数内的 `test_block` 的结果由ioloop 内send



## _handler 启动时的注册

main.py application.listen(xx) 

web.py listen

tcpserver.py listen

socket.bind  `AF_INET` 和 `AF_INET6` 的2个 socket  在指定端口建2个socket 

顺便学习下socket

```python

af, socktype, proto, canonname, sockaddr = res

## AF_INET is the address family that is used for the socket you're creating (in this case an Internet Protocol address). The Linux kernel, for example, supports 29 other address families such as UNIX sockets and IPX, and also communications with IRDA and Bluetooth (AF_IRDA and AF_BLUETOOTH, but it is doubtful you'll use these at such a low level).

# For the most part sticking with AF_INET for socket programming over a network is the safest option.


## Stream Sockets: Delivery in a networked environment is guaranteed. If you send through the stream socket three items "A, B, C", they will arrive in the same order - "A, B, C". These sockets use TCP (Transmission Control Protocol) for data transmission. If delivery is impossible, the sender receives an error indicator. Data records do not have any boundaries.
# http://www.tutorialspoint.com/unix_sockets/what_is_socket.htm

## 第三个是protocol
#IPPROTO_IP = 0   system default
#IPPROTO_TCP = 6  tornado，（） 
# http://stackoverflow.com/questions/7346695/strange-linux-socket-protocols-behaviour

# Giving 0 as protocol to socket just means that you want to use the default protocol for the family/socktype pair. In this case that is TCP, and thus you get the same result as with IPPROTO_TCP.  orz

sock.listen(backlog)  # {int} 128

# The maximum length of the queue of pending connections

```

通过 netutil.py 注册到ioloop

```python

for sock in sockets:
    self._sockets[sock.fileno()] = sock
    add_accept_handler(sock, self._handle_connection,
                       io_loop=self.io_loop)
```

在这里 add-accept_handler 的时候就建立了与客户端的 socket `self._impl.register(fd, events | self.ERROR)`



### 那么与 web 层 的requestHandler 如何映射?

listener的端口，收到 epoll 的 event， 然后建立连接, 读数据, 正则匹配handler， 反射根据get,post等 调用相应 handler 然后如果handler 里面有io 则同上，如果是基于socket的 则由 epoll 通知，用thread 的则由 callback（带timeout的）通知。 

这里仅仅说 client 发起的纯 request 

web 层的
(httpserver.handle_stream)

读 header 是在 iostream.py 的 `_run_read_callback` 

读完header后，根据输入的正则，匹配handler 然后根据 header 数据调方法




## 总结

tornado 内还有很多东西，由于时间原因，以后再写吧。





## reference 

http://zoomq.qiniudn.com/ZQScrapBook/ZqFLOSS/data/20100927213110/

http://www.cnblogs.com/yiwenshengmei/archive/2011/06/08/understanding_tornado.html

https://artemrudenko.wordpress.com/2013/04/15/python-why-you-need-to-use-wraps-with-decorators/

http://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do-in-python

http://www.pengyi.info/tag/Tornado/?page=1

http://blog.xiaogaozi.org/2012/09/21/understanding-tornado-dot-gen/

http://dongweiming.github.io/Expert-Python/

http://www.slideshare.net/emptysquare/nyc-python-meetup-coroutines-2013-0416

