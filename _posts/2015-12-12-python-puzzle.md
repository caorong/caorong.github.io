---
layout: post
title: "python puzzle"
description: ""
category: "python"
tags: [python, puzzle]
published: true
---

记录python开发时遇到的一些令人眼前一亮的坑！

## python 对象重用

下面代码目的是构造2个 提供不同默认参数的 `testfn` 函数。正常的思路，如下。。

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

理想中的结果应该print出来2个函数是不同的...

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

sof的解决方法 [详](http://stackoverflow.com/questions/938429/scope-of-python-lambda-functions-and-their-parameters)

1. 加一层工厂，这样工厂里面的入参引用必定不同
2. 使用 functools.partial 建新函数

或者 循环创建函数的同时直接 消费或激活函数... (如果可以的话)

```python
for i in watch_list.keys():
    fn = functools.partial(testfn, name=i, data=watch_list[i])
    fn()
    print(fn, lambda: testfn(i, watch_list[i]))
```


## 用 dis 看python bytecode

比如

```py
a = 0
b = 0
a = a + 1
b += 1
```

```
  1           0 LOAD_CONST               0 (0)
              3 STORE_NAME               0 (a)

  2           6 LOAD_CONST               0 (0)
              9 STORE_NAME               1 (b)

  3          12 LOAD_NAME                0 (a)
             15 LOAD_CONST               1 (1)
             18 BINARY_ADD
             19 STORE_NAME               0 (a)

  4          22 LOAD_NAME                1 (b)
             25 LOAD_CONST               1 (1)
             28 INPLACE_ADD
             29 STORE_NAME               1 (b)
             32 LOAD_CONST               2 (None)
             35 RETURN_VALUE
```

所以 a的写法更好些。。。


