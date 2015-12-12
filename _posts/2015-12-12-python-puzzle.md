---
layout: post
title: "python puzzle"
description: ""
category: "python"
tags: [python, puzzle]
published: false
---

记录开发时遇到的一些令人眼前一亮的坑！

## lambda

```python

watch_list = {
    'a': 1,
    'b': 2,
}

def testfn(name, data):
    print(name, data)

for i in watch_list.keys():
    fn = functools.partial(testfn, name=i, data=watch_list[i])
    print(fn, lambda: testfn(i, watch_list[i]))
```

猜想下结果时什么？ 是不是print 出来的4个函数应该都是不同的？ 

然而...

```
(<functools.partial object at 0x10df9e208>, <function <lambda> at 0x10dfcf140>)
(<functools.partial object at 0x10dfd1310>, <function <lambda> at 0x10dfcf140>)
```

lambda 出来的函数竟然是一样的！！！

原因是lambda 里面存的是引用，也就是指针，这又牵扯python的一个设定，他会重用对象。

```python
In [10]: id('a')
Out[10]: 4328343392

In [11]: id('a')
Out[11]: 4328343392

In [12]: id('a')
Out[12]: 4328343392
```

于是上面的 lambda 对象也被重用了，于是发生这种结果。

解决方法 [详](http://stackoverflow.com/questions/938429/scope-of-python-lambda-functions-and-their-parameters)

1. 加一层工厂，这样工厂里面的入参引用必定不同
2. 使用 functools.partial 建新函数


## reference

