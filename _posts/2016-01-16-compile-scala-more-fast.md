---
layout: post
title: "用zinc 编译scala的测试"
description: ""
category: "scala"
tags: [scala, zinc]
published: true
---


zinc 可以增量编译scala，因为编译scala最慢的就是他自己的library了.

至于到底可以提高多少的编译速度(scala 自己的library 编译到底有多慢)，我做了个测试

## 使用方式

zinc -start

## 环境

java 7
java/scala 混合项目
maven

## 编译方式

mvn clean package (关闭除了scala-maven-plugin，compile-plugin以外所有的插件)

## 结果

no zinc server

[INFO] news-xxxx ................................ SUCCESS [ 30.615 s]
[INFO] news-xxx2 ................................ SUCCESS [ 59.325 s]


启动zinc 后 第一次

[INFO] news-xxxx ................................ SUCCESS [ 51.443 s]
[INFO] news-xxx2 ................................ SUCCESS [ 18.390 s]

第二次 编译，没有修改代码

[INFO] news-xxxx ................................ SUCCESS [ 19.443 s]
[INFO] news-xxx2 ................................ SUCCESS [ 17.214 s]

第三次 编译，没有修改代码

[INFO] news-xxxx ................................ SUCCESS [ 17.473 s]
[INFO] news-xxx2 ................................ SUCCESS [ 17.113 s]

修改几行代码后

[INFO] news-xxxx ................................ SUCCESS [ 22.295 s]
[INFO] news-xxx2 ................................ SUCCESS [ 19.668 s]


## 结论

效果不错，安利下，不过[idea用不了](http://blog.jetbrains.com/scala/2012/12/28/a-new-way-to-compile/) ＝ ＝ 





