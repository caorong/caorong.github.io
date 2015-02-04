---
layout: post
title: "绝不完全相信开源代码"
description: ""
category: "java"
tags: [ web, java, spring]
published: true
---

今天，公司开年会，然后下午就要走了，然后上午一下子发现上次迁移 db 的时候忘记搞双数据源了 ＝ ＝。于是，赶紧在一个小时内调代码。

代码写完了，感觉对了，一上线，立马错了！！！

### 错误是怎样的呢

先说下如何配置双数据源

简单来说，我要配置 2 个`datasource`， 因为我用的ibatis， 所以我还要配置2个`sqlmap`， 最后封装给spring ，于是 还有2个 `SqlMapClientTemplate`

于是我这样子搞的

```xml
<!-- 数据源1 -->

<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
destroy-method="close">
	<!-- 省略100行 -->
</bean>
<bean id="sqlMapClient" class="org.springframework.orm.ibatis.SqlMapClientFactoryBean">
	<property name="configLocation" value="classpath:sqlmap-config.xml" />
	<property name="dataSource" ref="dataSource" />
</bean>
<bean id="sqlMapp" class="org.springframework.orm.ibatis.SqlMapClientTemplate">
	<property name="sqlMapClient" ref="sqlMapClient" />
</bean>


<!-- 数据源2 -->

<bean id="dataSource2" class="com.mchange.v2.c3p0.ComboPooledDataSource"
destroy-method="close">
	<!-- 省略100行 -->
</bean>
<bean id="sqlMapClient2" class="org.springframework.orm.ibatis.SqlMapClientFactoryBean">
	<property name="configLocation" value="classpath:sqlmap-config2.xml" />
	<property name="dataSource" ref="dataSource2" />
</bean>
<bean id="sqlMapp" class="org.springframework.orm.ibatis.SqlMapClientTemplate">
	<property name="sqlMapClient" ref="sqlMapClient2" />
</bean>

```

看着没错吧，但一跑起来就错了。 sqlmapp的`dataSource`永远不是`dataSource2`。

因为很急，马上要走了，然后就觉得代码肯定没问题，是我的问题，尼玛。。。

回家后静下心调试了下

```java
System.out.println(dataSource);			// dataSource bean id = 11
System.out.println(dataSource2);		// dataSource bean id = 12

System.out.println(sqlMapClient.getObject().getDataSource().getConnection()); 	// dataSource bean id = 11
System.out.println(sqlMapClient2.getObject().getDataSource().getConnection());	// dataSource bean id = 12

System.out.println(sqlMap); 	// dataSource bean id = 11
System.out.println(sqlMapp);	// dataSource bean id = 11
```

好吧， 问题出在sqlMap， 即`org.springframework.orm.ibatis.SqlMapClientTemplate`，看了下代码


```java
public class SqlMapClientTemplate extends JdbcAccessor implements SqlMapClientOperations {

	private SqlMapClient sqlMapClient;

	public void setSqlMapClient(SqlMapClient sqlMapClient) {
		this.sqlMapClient = sqlMapClient;
	}

	@Override
	public DataSource getDataSource() {
		DataSource ds = super.getDataSource();
		return (ds != null ? ds : this.sqlMapClient.getDataSource());
	}
}
```

再看他的super class ＝> JdbcAccessor

```java
public abstract class JdbcAccessor implements InitializingBean {

	private DataSource dataSource;

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	/**
	 * Return the DataSource used by this template.
	 */
	public DataSource getDataSource() {
		return this.dataSource;
	}
}
```

好吧，他直接根据bean name， 直接set了`DataSource`，但尼玛这代码怎么封装的这么蛋疼！！

我的理解，既然`sqlMapClient`， 已经配置了`DataSource`为什么我把`sqlMapClient`set到`SqlMapClientTemplate`的时候他应该这么写

```java
public class SqlMapClientTemplate extends JdbcAccessor implements SqlMapClientOperations {

	public void setSqlMapClient(SqlMapClient sqlMapClient) {
			this.sqlMapClient = sqlMapClient;
			this.dataSource = sqlMapCient.getDataSource();
	}
}
```

表示他竟然强制逼我再在他里面配置一便`DataSource`，真是醉了。。。。


好吧，总结下，写代码，切勿急躁 以及，有些代码封装的真的很蛋疼，一定要看源码。



