---
layout: post
title: "编译 apue2e 和 apue3e 代码"
description: ""
category: "learning"
tags: [apue,unix,YouCompleteMe,vim]
---

最近觉得一直写高层语言写的有点淡然无味了，遂拿起apue开始看起来，调和一下。


看到第一个例子的时候就看到代码 include 了作者自己的库，`apue.h` 然后下载源码后编译了几次才过，记录一下。。


###apue 2e

首先看README，需要根据系统更改Make.defines.{your system},的WKDIR，

然后先别make all

不然会遇到错误

```text
clang: warning: argument unused during compilation: '-ansi'
Undefined symbols for architecture x86_64:
  "_CMSG_LEN", referenced from:
      _recv_fd in libapue.a(recvfd.o)
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make[2]: *** [call] Error 1
make[1]: *** [macos] Error 1
make: *** [all] Error 2
```

然后到apue.2e/

```c
#if defined(SOLARIS)
#define _XOPEN_SOURCE	500	/* Single UNIX Specification, Version 2  for Solaris 9 */
#define CMSG_LEN(x)	_CMSG_DATA_ALIGN(sizeof(struct cmsghdr)+(x))
#elif !defined(BSD)
#define _XOPEN_SOURCE	600	/* Single UNIX Specification, Version 3 */
#endif

#ifdef MACOS
#define _DARWIN_C_SOURCE
#endif
```

需要加上最后3行，至于为啥详见[这里](https://developer.apple.com/library/mac/documentation/Darwin/Reference/Manpages/man5/compat.5.html)

```text
With _POSIX_C_SOURCE or _DARWIN_C_SOURCE for i386, or when building for any other architecture, system and library calls conform to Version 3 of the Single UNIX Specification (``SUSv3'').
```

编译完apue的lib后，如此编译代码：

```sh
clang -Wall -I apue.2e/include/ ls1.c apue.2e/lib/libapue.a -o ls1
```

- - -

###apue 3e

话说下载源码的时候，看到官网上出现了第三版，于是果断下载了电子书，看看

话说第三版编译超级方便，直接make就行了。

```text
On Freebsd, type "gmake".
On other platforms, type "make" (as long as this is gnu make).
```

但是。。。

又出现了这个报错

```sh
clang: error: unknown argument: '-R.' [-Wunused-command-line-argument-hard-error-in-future]
```

去`apue.3e/db/`把它的makefile里面的－R去掉就行了。话说不管是装gem还是pip装python lib都不定期遇到这个问题，不过这次前置AFLAGS不知为何无效。clang认为他用不到的，你敢写？！

```makefile
# line20
ifeq "$(PLATFORM)" "macos"
  # EXTRALD=-R.
  EXTRALD=
endif
```

然后就顺利编译完成，开始看书，写代码吧。

- - -

最后写代码推荐如果用vim的话，一定要配合https://github.com/Valloric/YouCompleteMe

附[.ycm_extra_conf.py](https://gist.github.com/caorong/1ba46227534c3473257a)

谁用谁知道，自带语法检测，代码跳转，包办了syntastic，ctags的功能，极其强大！

