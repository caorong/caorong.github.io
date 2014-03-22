---
layout: post
title: "开始用jetty调代码"
description: ""
category: "work"
tags: [java,tomcat,jetty,web，eclipse]
---
{% include JB/setup %}

今天我研究了一上午，尝试了下用jetty配合eclipse调试web项目。

###why choose jetty
迷之音：eclipse本来不就有server插件的嘛，可以直接用tomcat呀，为嘛要用jetty呀？

嗯哼，前几天调试代码的时候，有不知道那个文件被锁住还是神马的，class没有拷贝到eclipse的temp tomcat目录里面。害得我又浪费时间，以为是小伙伴的代码出了问题，各种看别人代码。。。

###why tomcat with eclipse sometimes make me crazy
迷之音：eclipse的workplace里面项目多了以后好像是经常性抽筋额。是神马原因造成的呢？

嗯哼，我就说说maven的web项目，在eclispe里面每次按下`ctrl+s`保存代码时maven会把编译好的代码放到target里面，然后如果配置了server的话，还会编译一份到eclipse的workplace里面的某号temp server里面比如
	
`X:\project_workplace\.metadata\.plugins
\org.eclipse.wst.server.core\tmpx\m2ewebapp\WEB-INF\classes`
	
嗯，一份代码，竟然需要复制到2个地方。我猜可能是内存不够或者某原因文件被锁。so，总之没复制全，导致xxclass not found，或者别的类似错误。

那如果我用jetty的话，会怎样？我只要指定我的webapp的path是target目录，然后就不用管他了，也就没有了所谓的第二次的复制，异常发生机率更小了，即使发生异常，也只要delete掉target，然后rebuild就行了，不像eclipse还要clean server，有时clean server还会报exception，真让人想掀桌！！！

###config jetty plugin in maven
迷之音：原来如此，那就快教教我怎么配置maven的eclipse插件啊

1.在pom里面的 `<build>` 下加一个plugin

{% highlight xml %} 
 
<plugin>
	<groupId>org.eclipse.jetty</groupId> <!-- if jdk7 jetty9 -->
	<groupId>org.mortbay.jetty</groupId> <!-- if jdk6 jetty8 -->
	<artifactId>jetty-maven-plugin</artifactId>
	<configuration>
		<webApp>
			<descriptor>WebContent/WEB-INF/web.xml</descriptor>
		</webApp>
	</configuration>
</plugin>
{% endhighlight %}

关于版本的选择

如果你是jdk7的话，那你可以用最与时俱进的jetty9啦，groupid用eclipse，不加version自动下载最新的版本，因为他是compiled by jdk7 所以不支持jdk6 :p


[ org.eclipse.jetty's maven仓库](http://repo.maven.apache.org/maven2/org/eclipse/jetty/jetty-maven-plugin/)

由于我还在用苦逼的6，groupid得用org.mortbay.jetty ，jetty8

[org.mortbay.jetty's maven仓库](http://central.maven.org/maven2/org/mortbay/jetty/jetty-maven-plugin/)



2.将编译的class的目标目录改到WebContent/WEB-INF/class下面(也可以通过改eclipse的project的build的target，改pom的话，在家用idea继续写代码就不用再改啦)
`<outputDirectory>WebContent/WEBINF/classes</outputDirectory>`

3.mvn jetty:run 起动



####reference

http://www.eclipse.org/jetty/documentation/current/jetty-maven-plugin.html

http://blog.log4d.com/2011/04/run-jetty-in-maven/

http://stackoverflow.com/questions/10426557/missing-maven-plugin-jetty
