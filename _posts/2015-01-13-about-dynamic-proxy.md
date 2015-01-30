---
layout: post
title: "扯一下动态代理"
description: ""
category: "java"
tags: [ java, scala, redis, with]
published: true
---

表示好久没写blog了啊，上次写已经是去年的事情了啊！！！

还是自己的项目, 不过这次用到了redis，然后用到了jedis，然后用到了pool，于是得想个简单的方式封装下

### 为什么要封装

因为如果不封装，那么如果用connection pool的话，那么代码需要这么写


```java
ShardedJedisPool pool = new ShardedJedisPool(new Config(), shards);
ShardedJedis jedis = pool.getResource();
jedis.set("a", "foo");
.... // do your work here
pool.returnResource(jedis);
.... // a few moments later
ShardedJedis jedis2 = pool.getResource();
jedis.set("z", "bar");
pool.returnResource(jedis);

// before application exit
pool.destroy();
```

也就是说每一句对 redis 的操作我前后都要加上 pool.getResource 和 pool.returnResource 

嗯，这么干显然是太2了

### 关于pool

pool 帮我干了这些事情

1. 建立了 n 个没有 close 的 connection

然后还有一些管理 connection 的策略，然后，没了。至少 jedis 的源码就是这么简单, 表示他直接用了apache pool， 没有定义额外功能。

于是，如果光拿 connection 不还，则会导致第超过 MaxCount 个请求想要连接的时候就会 block 住，直到别人还 connection 。。。


### 正题 => 封装

怎么封装，以及`封装者`与`使用者`的感受。。。

#### java的传统封装方式 － 动态代理 － 用的爽，封的蛋疼

表示这算是一种正统的封装方式。原理就是动态调用method，并在前后增加重复逻辑。

但是由于封装出的对象和需要代理的对象`必须出自一个接口` 

但是

```java
public class Jedis extends BinaryJedis implements JedisCommands,
  MultiKeyCommands, AdvancedJedisCommands, ScriptingCommands,
  BasicCommands, ClusterCommands {
    // ...
  }
```

jedis implements 太多的接口，尼玛！


方法1. 残缺方案，（仅仅封装用的到的）比如JedisCommands

```java
public JedisCommands redis() {
    return (JedisCommands) Proxy.newProxyInstance(this.getClass().getClassLoader(),
      new Class[]{JedisCommands.class}, new InvocationHandler() {
          @Override
          public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
              Jedis j = RedisInner.pool().getResource();
              try {
                  return method.invoke(j, args);
              } catch (Exception e) {
                  throw e.getCause();
              }finally {
                  RedisInner.pool().returnResourceObject(j);
              }
          }
      });
}

public static void main(String[] args) {
    JedisCommands s = new Rredis().redis();
    System.out.println(s);
    System.out.println(s.set("aa", "13"));
    System.out.println(s.get("aa"));
}

```


不过这样的话，别的指令不能用了。

方法2. 方法1的扩展，既然 jedis 继承了6个接口，那我封6个代理对象不就得了！

（你愿意我也不拦你）

方法3. 究极方案(也是最烦的)，自己再封装一个 Jedis class 和 interface，囊括那6个。

（要写好多好多代码，你愿意我也不拦你）


#### python 的封装方式 with (举例子而已..)

```python
from contextlib import contextmanager

@contextmanager
def jedis():
    j = RedisInner.pool.getResource
    try:
      yield j
    finally:
      RedisInner.pool.returnResourceObject(j)

with jedis() as j:
  j.set("aa","12")

```

这个封装的很爽，只不过每次都要包 with


表示 scala 也可以用 with！！！


#### scala's with 

参考了[这个](https://github.com/pk11/sedis/blob/master/src/main/scala/sedis.scala#L100) 

```scala

def withJedis[T](body: Jedis => T): T = {
    val jedis: Jedis = RedisInner.pool.getResource
    try {
      body(jedis)
    } finally {
      RedisInner.pool.returnResourceObject(jedis)
    }
  }

withJedis { c =>
  println(c.set("aa", "1"))
  println(c.get("aa"))
}
```

表示一样好用到爆啊！！！ 我自己项目就用的这种


还看到一种 封装者要花点力气的方案 （不是花一点点）

https://github.com/top10/scala-redis-client/blob/master/src/main/scala/com/top10/redis/SingleRedis.scala

表示写了 200+行代码其实就代理了 JedisCommands 和 BasicCommands 2个接口，好吧，你是劳模


最后，我蛋疼的将 java create proxy 的代码硬是翻译成了 scala， 又发现一个坑！！

```scala
def jedis(): JedisCommands = {
    java.lang.reflect.Proxy.newProxyInstance(
    Redis.getClass.getClassLoader,
      Array(classOf[JedisCommands], classOf[BasicCommands]),
      new InvocationHandler {
        override def invoke(proxy: Object, method: Method, args: Array[Object]): Object = {
          val j = RedisInner.pool.getResource
          try {
            return method.invoke(j, args:_*)
          } catch {
            case e: Exception => {
              logger.error(e.getMessage, e)
              throw e.getCause
            }
          } finally {
            RedisInner.pool.returnResourceObject(j)
          }
          null
        }
      }
    ).asInstanceOf[JedisCommands]
  }

def main(args: Array[String]): Unit = {
  println(jedis().set("aa", "14"))
  println(jedis().get("aa"))
}
```

对比下这个, 表示数组入参又是一神坑！也是醉了

```scala
// java
method.invoke(j, args);
// scala
method.invoke(j, args:_*)
```

```The invoke method takes a varargs parameter. In Scala (unlike Java), you can't just pass an array and have it automatically treated as all the arguments```

可以参考下[这里](http://stackoverflow.com/questions/4871757/wrong-number-of-arguments-invoking-a-scala-constructor-using-reflection?rq=1)




