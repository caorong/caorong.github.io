---
layout: post
title: "debug Clojure"
description: ""
category: "Clojure"
tags: [Clojure, java, schemer]
published: true
---


## 调试clojure程序的几个方法

1. 打 print

2. 用ide 的debug功能 比如 idea

3. 用工具 [tools.trace](https://github.com/clojure/tools.trace) 打印堆栈信息

4. 配合上面的远程调试repl

下面主要讲4

因为我的idea经常死机，所以我自己是先用 lein 启动一个 repl，然后用 idea 连接到 remote repl。[参考 cursive 的 RemoteREPL](https://cursiveclojure.com/userguide/repl.html)


因为 lein 会在启动的时候先读取自定义参数

```sh
# Support $JAVA_OPTS for backwards-compatibility.
JVM_OPTS=${JVM_OPTS:-"$JAVA_OPTS"}
JAVA_CMD=${JAVA_CMD:-"java"}

```

所以可以启动之前先加入远程调试参数

```sh
$ export "JAVA_OPTS=-Xdebug -Xrunjdwp:transport=dt_socket,address=9092,server=y,suspend=n -Dclojure.compiler.disable-locals-clearing=true"

$ lein repl
```

最后, ide 远程连repl，和远程debug，就可以调试repl了。

#### 注意 

现在的 tools.trace 如需打印堆栈信息需要自己再函数前面加上 `^:dynamic`

```Clojure
(defn ^:dynamic fib [n] (if (< n 2) n (+ (fib (- n 1)) (fib (- n 2)))))
(dotrace [fib] (fib 3))
```


#### refrence

http://stackoverflow.com/questions/2352020/debugging-in-clojure

https://github.com/acug/drinkup/issues/1