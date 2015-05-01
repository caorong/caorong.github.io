---
layout: post
title: "java系 热部署"
description: ""
category: "tech"
tags: [java,hotswap]
published: true
---

之前有段时间一直在找 jreble 的免费替代方案。

### 用jetty自带的scan

hot swap
https://gist.github.com/naaman/1053217


### 给jvm打补丁

主要用的 [dcevm](https://github.com/dcevm/dcevm/releases) ＋ [HotswapAgent](https://github.com/HotswapProjects/HotswapAgent)

打补丁方法参考 http://ssw.jku.at/dcevm/binaries/

然后再搭配hotswapAgent启动

不过这个由于不能支持main方法启动的jetty server，所以放弃了，tomcat我自己没测过。


### [spring-loaded](https://github.com/spring-projects/spring-loaded)

spring出品，貌似grails就用的这个。不用给jvm打补丁，只要加一个agent。

至少我目前测试了main方法启动的jetty，和tomcat。

tomcat的话需要记得修改配置文件 (server.xml)，将reloadable 设置为false。

```sh
java -javaagent:/Users/xxxx/Documents/springloaded-1.2.1.RELEASE.jar -noverify SomeJavaClass
```


## 注意事项

机器性能顶得住的话可以将 IDEA 弄成和elipse一样 ctrl＋s 就能编译 [参考这里](https://groups.google.com/forum/#!topic/hotswapagent/BxAK_Clniss)



## reference

http://javainformed.blogspot.jp/2014/01/jrebel-free-alternative.html

http://stackoverflow.com/questions/7998669/redeploy-alternatives-to-jrebel

http://stackoverflow.com/questions/15118681/intelij-tomcat-spring-loaded

http://padcom13.blogspot.jp/2012/12/did-you-noticed-spring-loaded-is-here.html

http://vitalflux.com/configure-springloaded-eclipse-dynamic-web-project/

