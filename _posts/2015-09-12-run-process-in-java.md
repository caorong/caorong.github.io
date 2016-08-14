---
layout: post
title: "java process 的坑"
description: ""
category: "java"
tags: [java, process, ffmpeg]
published: true
---

最近把线上的java 升级到java7， 跑爬流的process，结果引起了一系列问题，随后几天加班加点解决问题，硬逼着了解了Process 的内幕，这里做下记录。


我们的java程序主要干的事情是通过启动外部进程 `FFmpeg` 来爬网络流，然后对它切分片。原本是跑在jdk6上，现在则是迁移到了jdk7。

随后先后引发了一系列问题，`too many open files`, `Process 关不掉` 等问题。

- 那么为什么要升级到java7？

    java7 支持能利用linux的 inotify 的通知（分片生成），之前java6 时是自己定时扫描。目的是为了减少io, 降低负载.

- 大致架构

    简单来说就是一个网络流我们启一对Thread，一个是在Process里面启动一个 FFmpeg，然后block在那里，另一个是对这个流切出来的分片做xxx业务，如下图

```text
+---------------------------+
|jvm                        |
| +-----------------------+ |
| |Thread                 | |
| |  +--------+---------+ | |
| |  |process | ffmpeg  | | |
| |  +------------------+ | |
| +-----------------------+ |
|                  * n      |
| +-----------------------+ |
| |Thread                 | |
| | +-------------------+ | |
| | |WatchService       | | |
| | +-------------------+ | |
| +-----------------------+ |
|                  * n      |
+---------------------------+
```

- too many open files

    先是上了一台机器看下情况，结果发现启动后爆出大量 open files，我们是一台机器, 一个jvm 启动 200左右的 FFmpeg 进程。测试环境没什么问题，(测试环境只启动了60+个FFmpeg) ，于是在测试环境也改成启动200个 FFmpeg，同样出现了这个问题。

    因为升级之前和之后，数量没有变化，所以应该和Process的数量无关，也就是说也许和我们新使用的WatchService 有关。而 jdk7的 WatchService使用的是系统的 inotify ，于是google了下，发现是因为超过了系统的最大限制, [解决方法如下](http://unix.stackexchange.com/questions/13751/kernel-inotify-watch-limit-reached)。 

- Process 无法关闭

    不过系统长时间运行后，发现启动了很多个相同的FFmpeg进程。当然原因是业务逻辑中有自动重启不良流的逻辑，并且如果重启有个超时时间。超过超时时间后会放弃关闭之前一个Process，强制启动一个新的Process。

    所以问题的关键，为什么会关不掉。

    首先在Linux里面启动一个 [Process](http://www.linuxprocess.com/process_basic/README.html) 会生成3个pipe，与你的shell交互，分别是input，output，error。

    java 启动一个process 怎么与上面的3个pipe关联 

    ```java
    // 
    Process pro = new ProcessBuilder(Arrays.asList(commandParamsArray)).
                    redirectErrorStream(true).start();
 
    // ---------------------

    // new ProcessBuilder 内部会调用native fork一个子进程，并获得native 写入的fds数组。
    pid = forkAndExec(launchMechanism.value,
                          helperpath,
                          prog,
                          argBlock, argc,
                          envBlock, envc,
                          dir,
                          fds,
                          redirectErrorStream);

    // ---------------------

    // native 启动外部进程成功后的回调
        try {
            doPrivileged(new PrivilegedExceptionAction<Void>() {
                public Void run() throws IOException {
                    initStreams(fds);
                    return null;
                }});

    // 将 3个 pipe 抽象成java里面的stream ，对应为 stdin, stdout, stderr
    void initStreams(int[] fds) throws IOException {
        stdin = (fds[0] == -1) ?
            ProcessBuilder.NullOutputStream.INSTANCE :
            new ProcessPipeOutputStream(fds[0]);

        stdout = (fds[1] == -1) ?
            ProcessBuilder.NullInputStream.INSTANCE :
            new ProcessPipeInputStream(fds[1]);

        stderr = (fds[2] == -1) ?
            ProcessBuilder.NullInputStream.INSTANCE :
            new ProcessPipeInputStream(fds[2]);

        // 启动一个thread， 在外部进程退出后执行关3上面3个stream
        processReaperExecutor.execute(new Runnable() {
            public void run() {
                // native 的函数，会block
                int exitcode = waitForProcessExit(pid);
                UNIXProcess.this.processExited(exitcode);
            }});
    }

    // 关闭流，以免资源泄漏
    void processExited(int exitcode) {
        synchronized (this) {
            this.exitcode = exitcode;
            hasExited = true;
            notifyAll();
        }

        if (stdout instanceof ProcessPipeInputStream)
            ((ProcessPipeInputStream) stdout).processExited();

        if (stderr instanceof ProcessPipeInputStream)
            ((ProcessPipeInputStream) stderr).processExited();

        if (stdin instanceof ProcessPipeOutputStream)
            ((ProcessPipeOutputStream) stdin).processExited();
        }
    }
    ```

    总结来说就是 java 从 native 获得了3个pipe。 写个测试证实一下

    ```text
    caorong@caorong-OptiPlex-3020 /proc/11140/fd
    $ ll |grep pipe
    lr-x------ 1 caorong caorong 64  9月 14 09:31 0 -> pipe:[47002731]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 1 -> pipe:[47002732]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 128 -> pipe:[47005726]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 129 -> pipe:[47005726]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 131 -> pipe:[47005727]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 132 -> pipe:[47005727]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 134 -> pipe:[47005728]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 135 -> pipe:[47005728]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 138 -> pipe:[47002777]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 139 -> pipe:[47002777]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 146 -> pipe:[47003850]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 148 -> pipe:[47003851]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 150 -> pipe:[47003852]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 2 -> pipe:[47002733]
    lr-x------ 1 caorong caorong 64  9月 14 09:31 38 -> pipe:[47002566]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 39 -> pipe:[47002566]


    ffmpeg

    lr-x------ 1 caorong caorong 64  9月 14 09:31 0 -> pipe:[47003850]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 1 -> pipe:[47003851]
    l-wx------ 1 caorong caorong 64  9月 14 09:31 2 -> pipe:[47003851]

    ```

    可以发现 ffmpeg 的fd有3个pipe，而且和这3个在java 的fd里有相同的。

    仔细看可以发现pipe1，和2 是相同的，这时因为我代码里 redirectErrorStream 了。

    也就是说在fork的时候就干了这件事情，（从中可以知道，为什么用了redirect之后，errorstream可以不消费了的原理）


    于是可以总结出下面一张图
    
    ```text
                         stdin               stdin             
                        +------+            +--+               
    +----------------+  |stdout   +-----+   |stdout +---------+
    | java process   | <-------+  | sys | <----+    |ffmpeg   |
    +----------------+  |stderr   +-----+   |stderr +---------+
                        +------+            +--+               
                                                               
                         pipe               pipe               
    ```
    
    java 和 他的子进程是通过一个pipe获得子进程的输出的。

    之前走进过一个误区，以为只要我保证kill掉java这边的3个stream 就行了，至少能让这个子进程游离脱离java，ppid变成1， 其实不然，ppid 仍然是java的pid，由图可知，这么做只是关掉java这端的pipe，于是ffmpeg生成的stdout 没人消费了，于是堆积在pipe，反而会造成内存泄漏。


    于是最终解决方案就是一个，kill掉ffmpeg

    这时，ted灵机一动，想出了多destory 几次process的方法，最多kill 100次，100次后再失败就不管了，经过10小时睡眠后发现问题没有复现。

    找了了下ffmpeg源码发现

    ```c
    static void
    sigterm_handler(int sig)
    {
        received_sigterm = sig;
        received_nb_signals++;
        term_exit_sigsafe();
        if(received_nb_signals > 3) {
            write(2/*STDERR_FILENO*/, "Received > 3 system signals, hard exiting\n",
                               strlen("Received > 3 system signals, hard exiting\n"));

            exit(123);
        }
    }
    ```

    ...


## Process 的最佳实践

1. 一定要保证启动的Process能关掉。

    如果自己无法保证，最好外面套一层watch Dog， 如果watchDog 还是kill不掉(watchDog 并不kill －9， destory 默认是 -15), 建议重写watchDog的 timeoutOccured 方法

    ```java
    @Override
    public synchronized void timeoutOccured(Watchdog w) {
        super.timeoutOccured(w);
        // only useable if sys destroy fail
        if (pid != null) {
            try {
                Runtime.getRuntime().exec("kill -9 " + pid);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```

2. 避免资源泄漏

    这里其实jdk已经帮你干了很多，知道保证外部process退出，就一定能回收所有的stream。

    不过，注意消费掉 外部process的stdout 和stderr，一个tips 可以用processBuilder的redirectErrorStream。

    ```java
    ProcessBuilder(Arrays.asList(commandParamsArray)).
                redirectErrorStream(true).start();
    ```

    然后在一个Thread 里消费stream。

    如果你的业务是需要长时间启动一个Process，比如我们这种，那么不要让stream的error影响到你的业务。

    如果你的业务知识启动一个短期的Process，并且不希望资源泄漏，请在stream 里遇到异常的时候 destory 掉 process;

    ```java
    try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(errorStream, "UTF-8"));
            String line;
            while ((line = reader.readLine()) != null) {
                // 日志 照常输出
                if (log.isDebugEnabled())
                    log.debug(line);
            }
        } catch (IOException e) {
            log.error("read from process error ", e);
            // process.destory();
            throw e;
        } finally {
            try {
                errorStream.close();
            } catch (IOException e) {
                log.error("close errorStream error", e);
            }
        }
    ```

## 为什么升级到jdk7之后问题显示出来了

TODO



## reference 

http://unix.stackexchange.com/questions/13751/kernel-inotify-watch-limit-reached
