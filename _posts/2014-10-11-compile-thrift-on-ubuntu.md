---
layout: post
title: "ubuntu上 编译 thrift"
description: ""
category: "linux"
tags: [linux, ubuntu, thrift]
---

写代码时需要生成thrift 代码，不巧最近系统刚换了ubuntu，于是便琢磨着编译了下thrift。又遇到了写坑。。。

====

首先，我目前编译的 thirft 只需要支持 java， ruby 和 python 


debian下预先需要安装的 lib [见官方](http://thrift.apache.org/docs/install/debian)

cpp搞不定，去除掉 `./configure --without-cpp`

编译python 没什么问题

编译java 需要注意的是如果是自己装的jdk 非apt装的，试试看 sudo java能否运行  参考 http://superuser.com/questions/709515/command-not-found-when-using-sudo

编译ruby ，先删除rbenv，然后apt-get install ruby ruby-dev

然后清除 gem 中已经存在的依赖， 根据gemspec => `/home/xxx/tools/thrift-0.9.1/lib/rb/thrift.gemspec` ，以免因为半本问题，比如rspec的新版本导致make时测试过不了。

"~>" 这个符号的表明 依赖的是这个版本 或者比这个版本高一个最小版本的gem [参考stackoverflow](http://stackoverflow.com/questions/5170547/what-does-tilde-greater-than-mean-in-ruby-gem-dependencies)

而如果你已经有较高的版本的话，他默认用较高的版本，
于是曾经因为 rspec 的大版本相差1，test怎么都过不了。 
（注，make 和 make install ruby都会跑一次 rspec 单元测试）


最后实在不行 就装预编译好的吧 0.9.1的 http://packages.ubuntu.com/utopic/thrift-compiler





