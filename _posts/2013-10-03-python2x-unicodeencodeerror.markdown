---
layout: post
title: "python2x UnicodeEncodeError"
tagline: "python 2.* 处理中文时遇到的问题"
description: "-desc-"
category: python
tags: [unicode, python, 中文]
---
 

当用python处理中文时经常会遇到这个问题，baidu google也能搜到铺天盖地的解决方法，其中搜出来最多的是如下方法。

```python
import sys
reload(sys)
sys.setdefaultencoding('utf8')
```

那为什么如此这般就行了？

先搞一个概念，一般写python时，创建文件后，开头总喜欢加一句

```python
#coding:utf8
print '豆奶'
```

这个表示该文件，__注意__，表示当前文件，比如xx.py这段代码用是以utf-8编码的。这和上面的setdefaultencoding是两码事！如果不加这一行，(python２x的文件编码默认也是ascii)但是代码中有中文出现的话，比如“豆奶”，则也会报同样的异常，原因是，“豆奶”这２个字符在ascii码里找不到，即无法encode成ascii字符。

而比如我现在用request爬一个web页面，当我把页面相关信息写入文件或者存入数据库时会发生标题所发生的错误。又是什么原因？

```python
# coding:utf8
import requests,sys

def writefile(name,msg):
    fp = open(name,'a')
    fp.write(msg)
    fp.flush()
    fp.close()

if __name__=='__main__':
	url = 'http://www.xxx.com'
	response = requests.get(url)
	response.encoding='utf-8'
	print response.text    	
	print response.encoding   
	print type(response.text)        # <type 'unicode'>
	writefile('test.txt',response.text) #UnicodeEncodeError
```

先说一下解决异常的方法。
就我所只有２种方法:
1. google出来最多的，在代码开头加上

```python
import sys
reload(sys)
sys.setdefaultencoding('utf8')
```

2. 在__仅接受编码过的string__的操作，比如存数据库，写文件。所以在进行这个操作的时候手动编码。

```python
def writefile(name,msg):
    fp = open(name,'a')

    #just leave the character out of the Unicode result　
    fp.write(msg.encode('utf8','ignore'))  
    fp.flush()
    fp.close()
```

#####总结：
因为response.text 的type是unicode，而python对于不接受unicode的xx操作，默认进行ascii编码。所以造成了如上错误。

顺便贴一下此默认设置的出处的搜索方法和源码。我的环境是mac。一般在python的lib下,我用ack依样画葫芦search。

```sh
ack -i setdefaultencoding /Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7

/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/site.py
504:        sys.setdefaultencoding(encoding) # Needs Python Unicode build !
557:    # Remove sys.setdefaultencoding() so that users cannot change the
560:    if hasattr(sys, "setdefaultencoding"):
561:        del sys.setdefaultencoding
```


```python
def setencoding():
    """Set the string encoding used by the Unicode implementation.  The
    default is 'ascii', but if you're willing to experiment, you can
    change this."""
    encoding = "ascii" # Default value set by _PyUnicode_Init()
    if 0:
        # Enable to support locale aware default string encodings.
        import locale
        loc = locale.getdefaultlocale()
        if loc[1]:
            encoding = loc[1]
    if 0:
        # Enable to switch off string to Unicode coercion and implicit
        # Unicode to string conversion.
        encoding = "undefined"
    if encoding != "ascii":
        # On Non-Unicode builds this will raise an AttributeError...
        sys.setdefaultencoding(encoding) # Needs Python Unicode build !
```


代码都是运行不到的，主要看注释。。当然可以手动修改

```
encoding = 'utf-8' 
```

一劳永逸。这也就是为什么google说也能通过修改site.py 的 encoding达到目的。

参考：

<http://www.iteye.com/topic/561786>

<http://www.iteye.com/topic/699510>

<http://docs.python.org/2/howto/unicode.html>
