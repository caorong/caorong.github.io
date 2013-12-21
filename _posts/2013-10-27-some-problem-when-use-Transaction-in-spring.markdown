---
layout: post
title: "使用spring @Transactional时遇到的问题"
tagline: "关于java的代理模式"
description: "-desc-"
category: java
tags: [java, spring, Transactional]
---
{% include JB/setup %}


这周在抄袭ruby代码时遇到了个奇怪的问题。

原代码
{% highlight java %}
public class XXService implements IXXService {

	@Override
	@Transactional
	public void Dojob(xxx input) {
		// operate database
		..
		// send queue
		sendqueue(msg);
	}
}
{% endhighlight %}

后面发现由于send queue时，收queue方也会对数据库操作，并发比较厉害时会发生不一致问题。so 将send queue的部分与 operate database 部分分开

新代码
{% highlight java %}
public class XXService implements IXXService {
	
	@Override
	public void Dojob(xxx input) {
		this.operatedb();
		// send queue
		sendqueue(msg);
	}
	
	@Override
	@Transactional
	public void operatedb(xxx input) {
		// operate database
		..
		// send queue
		sendqueue(msg);
	}
}
{% endhighlight %}

####看起来还是挺合理的代码额，然后单步调试时发现事务并没有起作用，可是改之前事务明明是器作用的呀？

参考springsource reference，可并没有找到什么。。。

然后，考虑到java的编码范式，将Transaction的方法单独提出到一个新的service里。然后XXService里利用spring的autowire引入，
{% highlight java %}
public class XXService implements IXXService {
	
	@Autowired
	private XXDBService dbservice;
	@Override
	public void Dojob(xxx input) {
		dbservice.operatedb();
		// send queue
		sendqueue(msg);
	}
}

@Service
public class XXDBService implements IXXDBService {
	
	@Override
	@Transactional
	public void operatedb(xxx input) {
		// operate database
		..
		// send queue
		sendqueue(msg);
	}
}
{% endhighlight %}
结果。。it works!!!

但是为什么呢？说到这个其实与spring aop的实现有密不可分的联系。

那么spring aop是如何实现的呢？

这里暂不考虑第三方的aop库，仅用java官方提供的动态代理的方法。

首先需要了解下spring是如何创建一个dynamic对象的

同样抛开解析xml等等不谈。

他的本质其实就是java的动态代理：
{% highlight java %}
public class DoTransaction implements InvocationHandler{
	Object obj = null;
	public DoTransaction(Object _obj){
		this.obj = _obj;
	}
	@Override
	public Object invoke(Object proxy,Method method, Object[] args)throws Throwable{
		// 开启jdbc事务 (yy的，请勿较真)
		jdbc.transaction.start
		//执行自己的service里面的业务
		Object result = method.invoke(this.obj,args);
		// 提交事务 (yy的，请勿较真)
		jdbc.commit
		return result;
	}
}
{% endhighlight %}

所以，当我在一个方法前加了@Transactional。
于是，spring先在create 此 dynamic对象的时候将会实用DoTransaction创建一个proxy&xxx对象，此对象即带了事务该做的事情。

最后，再回到原命题，为神马this.operatedb()不起作用了呢？

那请问此this是神马？

this吗？ 

额。 

this，我debug看看此this是什么。

这个object名字里没带$proxy字段，这个this并没有被代理过，一个干干净净的object，so，在此类里面此@Transaction形同虚设了！！！

so why？

其实this自己已经是一个对象了，他做得事情其实就是goto到this.xx的函数里面，除非。。。

代码层面的aop？

好吧，java本质还是静态语言，至少我现在还没有发现能这么干。

so，还是把@Transaction的方法提到一个service里，以一个对象的形式引用吧。以便spring帮我们aop。

finally，嗯，奇怪的问题原来不过如此啊。。