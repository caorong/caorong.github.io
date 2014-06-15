---
layout: post
title: "最近遇到的几个坑"
description: ""
category: "work"
tags: [java,crawler,httpclient,windows,linux]
---

最近被几个坑搞死了，加了好多天班，一定要记录下来，备忘。

##### 小坑
在引入断点续传后，我发现最近爬虫经常在跑了2，3天后就再也下载不了东西了，所有的request都被阻塞住了，然后timeout。

原因是由于下载的代码是左抄抄，右抄抄，外加写以前的下载代码，然后忘记释放stream了。

需要释放request的releaseConnection 和 response的content的inputstream

```java
EntityUtils.consume(response.getEntity());
// or
request.releaseConnection();
```

##### 中坑
下载视频的时候有时候contentlength是拿不到的
看2个response的header

with content-length

```text
HTTP/1.0 206 Partial Content [Date: Thu, 12 Jun 2014 11:22:52 GMT, Server: nginx, Content-Type: video/mp4, Last-Modified: Tue, 27 May 2014 03:14:13 GMT, X-NG: gz-flv-29, Accept-Ranges: bytes, Vary: accept-encoding, Content-Range: bytes 0-18744342/18744343, Content-Length: 18744343, Age: 54372, Via: 1.0 jsycdx61:8104 (Cdn Cache Server V2.0), 1.0 xzh20:8888 (Cdn Cache Server V2.0), Connection: keep-alive]
```

without content-length

```text
HTTP/1.1 206 Partial Content [Server: nginx/6.2, Date: Fri, 13 Jun 2014 02:33:01 GMT, Content-Type: video/mp4, Last-Modified: Fri, 16 May 2014 22:20:14 GMT, Transfer-Encoding: chunked, Connection: keep-alive, Content-Range: bytes 0-20906947/20906948, Pragma: head=244109]
```
在httpClient里面，如果header里面没有Content-length则返回－1。

正常的cdn一般header里面都会带Content-length，有些坑爹的自建cdn处于某种理由就没带。

解法，length从Content-Range 里面取。

##### 大坑
最近在windows上开发，然后发现发到Linux 上后不停的报错。明明win上测试不报错的说。

```java
@Test
public void test() throws IOException {
    HttpClient client = new DefaultHttpClient();
    HttpGet request = new HttpGet("http://www.baidu.com");
    HttpResponse response = client.execute(request);
    request.releaseConnection();
    
    response.getEntity().getContent();
    System.out.println(IOUtils.toString(
            response.getEntity().getContent(),
            UrlUtils.getCharset(response.getEntity().getContentType().getValue())));
    EntityUtils.consume(response.getEntity());
}
```

这段代码其实有问题的。我不该在还未读取stream之前就把connection给释放掉。(因为是很早以前的代码，看都没看直接拿来用了)

然而这段代码在windows上跑是可以正常跑的，而换到linux/unix上就报`Socket closed`。

因为读流的操作都是java调用native的操作 

`at java.net.SocketInputStream.socketRead0(Native Method)`

而系统的操作我猜测是这样的，我让他execute一个request，然后它就读数据，然后放到某个交换区里面区。

然而，我releaseConnection后，windows和xnix的反应是不同的。

这里直接拿openjdk的源码来yy下

[windows的代码](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/windows/native/java/net/SocketInputStream.c#L74)

```c
//...
fd = (*env)->GetIntField(env, fdObj, IO_fd_fdID);
if (fd == -1) {
    NET_ThrowSocketException(env, "Socket closed");
    return -1;
}
```

[solaris 的代码](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/solaris/native/java/net/SocketInputStream.c#L75)

```c
} else {
    fd = (*env)->GetIntField(env, fdObj, IO_fd_fdID);
    /* Bug 4086704 - If the Socket associated with this file descriptor
     * was closed (sysCloseFD), the the file descriptor is set to -1.
     */
    if (fd == -1) {
        JNU_ThrowByName(env, "java/net/SocketException", "Socket closed");
        return -1;
    }
}
```

由此可见他们都先去拿了fd， 而对于已经释放了的资源，linux直接返回-1，而windows则可以拿到资源。

于是，后面又做了个实验，尝试下如果资源较大，在releaseconnection之前没有全放进交换区会怎样。

于是，找了个页面大小很大的网站，然后给java限速，测试发现这样的话，windows上跑也会报错了。

由此可猜测，windows的源码一定非常丑陋，一定充满了各种ifelse的判断。。。

