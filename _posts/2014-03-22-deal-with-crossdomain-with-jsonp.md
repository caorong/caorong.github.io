---
layout: post
title: "扯一下jsonp"
description: ""
category: "web"
tags: [js, ajax, jsonp, youku]
---
 

好久没有写blog了，最近实在是好忙啊。上周在研究[朱一的插件代码](https://github.com/zythum/youkuhtml5playerbookmark) 时，发现了原来可以用jsonp来解决跨域问题，而且解决方法非常巧妙，而且，优酷就是这么干的。

####首先，为什么会有跨域问题？

因为由浏览器的`同源策略`，所以从client端发出的ajax请求，只能请求这个网站的服务器。简单来说，就是ajax的request的前半部分是这个`http://www.currentsite.com/`。如果要请求别的域名下的服务，浏览器会弹错误信息。

####那么，关于script标签
script标签是个好东西，url里的domain是啥都行。正因如此，一些前端共用库比如jquery都会直接用cdn上的，一来提升用户的访问速度，二来节省了自己服务器的流量。比如[百度的共用库](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs)

####最后说说jsonp
所谓的jsonp就是利用script标签的无跨域问题的特性实现跨域请求。但需要服务端的配合。

####一个栗子
服务端代码，用于接收ajax请求，返回一个json：


```python
from bottle import route, request

json = "{\"a\" = \"1\"}"
callback = "callback("

@route('/data')
def get_data():
	if len(request.query['__callback']) != 0:
		callback = request.query['__callback'] + "("
	return callback + json + ")"

```

浏览器代码，
```javascript
<script url ="http://mydomain.com/data?__callback=run_callback" />
```

至此完结。

####script得到的是什么？
```javascript
run_callback("{\"a\" = \"1\"}")
// 哇，竟然将json传到某function里去了
```

```javascript
function run_callback(data){
	console.log(data['a'])
	// ...
```


是不是很巧妙！！！


####最后来个实际的栗子

话说其实优酷就是通过jsonp接口来跨域获取视频信息。

try it~

`http://v.youku.com/player/getPlaylist/VideoIDS/XNjc1MjQwNTQ4/Pf/4?__callback=abcdefg`

####最后有一个邪恶的想法
那利用这个接口+解真实地址的[方法](https://github.com/zythum/youkuhtml5playerbookmark/blob/master/2.0/source/youku.js)得到mp4格式的地址不就可以实现，无缝盗链了吗。。。

[邪恶想法实践](http://jsfiddle.net/lelouchcr/QyHvX/)，使用的是jquery封装的jsonp，直接播放优酷的视频without ad.
