---
layout: post
title: "spring Application里面的 bean 和 spring web 里面的 bean 会冲突吗?"
description: ""
category: "java"
tags: [java, spring, web]
---

之前在公司 gitlab 上看同事写的代码，其中在 spring-mvc.xml (就是配置在web.xml里的) 看到这么一段


``` xml
<!-- 注意事项请参考：http://jinnianshilongnian.iteye.com/blog/1762632 -->
<context:component-scan base-package="com.xxx.xxx.app.web" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
    <context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
</context:component-scan>
```

他的作用是使web的上下文仅扫描指定的注解。

然后后面的blog说明了一件我以前没想过的事情。就是不这么干的话有可能会使bean冲突。会让后面读取的用注解配置的 bean 覆盖了先初始化的bean（spring 中的 bean 在内存中用 beanname 标记）。

那么到底是谁覆盖了谁呢？

由于 web.xml 的加载顺序是 context-param -> listener -> filter -> servlet

所以配置 servlet 环境的bean将覆盖 其他 application－context 的 bean。

尼玛，我都写了一年的java，要不是意外看了下别人的代码，基本想不到这种问题，不过也是主要原因是我唯一写过的几个要对外提供 web 接口的项目都不需要事务或者需要对 service 做二次修饰操作。

不过话也说回来，如果项目用不到事务，用注解地方没有对注解的bean做aop等修饰，或者aop是卸载servlet的applicationContext里面的话，其实都不会造成错误的，只不过多初始化了一次。





