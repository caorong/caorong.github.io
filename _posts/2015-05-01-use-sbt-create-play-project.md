---
layout: post
title: "使用sbt 创建play工程"
description: ""
category: "scala"
tags: [scala, sbt, play]
published: true
---

比起[play官方](https://www.playframework.com/documentation/2.3.x/NewApplication)推荐的Activator, 作为从java转过来的开发者, 本来想用 maven 不过由于 play console 本身就是基于 sbt 开发的所以只能用sbt了

### sbt

sbt 底层是基于 `ivy` 的, `ivy` 是和 `maven` 类似的 jar包 依赖管理, 不过他们存放的目录结构有差异

所以使用 sbt 要解决2个问题: 

1. 如何复用原来 maven 本地仓库的jar

2. 修改远程仓库地址

上面两个问题都可以通过修改sbt的默认配置解决.

sbt 默认读取的配置是 sbt/bin/sbt-launch.jar:/sbt/sbt.boot.properties 

[sbt提供了2个方式](http://www.scala-sbt.org/release/docs/Proxy-Repositories.html)修改其默认配置

1. 不读取原来的配置

SBT_OPTS增加 `-Dsbt.override.build.repos=true` 并创建一个 `~/.sbt/repositories` 文件, 并修改其配置, 后面有我配置的内容

2. 重新指定配置

SBT_OPTS增加 `-Dsbt.repository.config=<path-to-your-repo-file>` 指定配置路径
比如 -Dsbt.repository.config=~/.sbt/repositories 达到和上面一样的效果


### 怎么修改SBT_OPTS

linux/macox 的话 直接 

```
$ env SBT_OPTS="-Dsbt.override.build.repos=true" sbt
```

或者配置在 `.bashrc/.bash_profile` 里面


或者[参考这里](http://stackoverflow.com/questions/3868863/how-to-specify-jvm-maximum-heap-size-xmx-for-running-an-application-with-run)


附上我的配置`~/.sbt/repositories`

```
[scala]
  version: ${sbt.scala.version-auto}

[app]
  org: ${sbt.organization-org.scala-sbt}
  name: sbt
  version: ${sbt.version-read(sbt.version)[0.13.8]}
  class: ${sbt.main.class-sbt.xMain}
  components: xsbti,extra
  cross-versioned: ${sbt.cross.versioned-false}
  resources: ${sbt.extraClasspath-}

[repositories]
  local
  local-Maven-Repository: file:///~/.m2/repository/jar, [organization]/[module]/[revision]/[type]s/[artifact](-[classifier]).[ext]
  # oschina-nexus: http://maven.oschina.net/content/groups/public/
  maven-central
  typesafe-ivy-releases: https://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly

[boot]
  directory: ${sbt.boot.directory-${sbt.global.base-${user.home}/.sbt}/boot/}

[ivy]
  ivy-home: ${sbt.ivy.home-${user.home}/.ivy2/}
  checksums: ${sbt.checksums-sha1,md5}
  override-build-repos: ${sbt.override.build.repos-false}
  repository-config: ${sbt.repository.config-${sbt.global.base-${user.home}/.sbt}/repositories}

```

oschina 有点坑, 由于最新的jar没同步, 所以各种超时


### play

然后等他下载依赖, 创建[play 目录结构](https://www.playframework.com/documentation/2.3.x/Anatomy)

因为原本的play-console就是sbt, 所以可以直接在sbt里面敲play 官方的所有命令

比如 [`playGenerateSecret`](https://www.playframework.com/documentation/2.3.x/ApplicationSecret)


### source

用maven的话, 如果在ide里面就勾上 auto download source 就可以让他自己下源码了.

可是这个规则在sbt里面行不通了, 并不是说sbt 不能下载源码而是play的依赖是`com.typesafe.play" % "sbt-plugin % "2.3.8"` 这个插件下载的. 貌似没办法设置让他里面下载jar的时候顺便下载源码.

后来看到[这篇blog](http://www.plotprojects.com/create-an-intellij-idea-project-with-library-sources-attached/)想到个变通的方式

直接在 `build.sbt` 里面对需要看源码的jar 手动加dependence

然后加一个alias, 打 `generate-project` 自动运行`update-classifiers`(下载source, doc) `gen-idea` (生成idea 项目) 

```
libraryDependencies ++= Seq(
  // "group" % "artifact" % "version"
  "com.typesafe.play" % "play_2.11" % "2.3.8",
  "com.typesafe.play" % "play-java-ebean_2.11" % "2.3.8"
)


addCommandAlias("generate-project", ";update-classifiers;gen-idea")
```

### idea

最后导入 idea 选择 SBT project, 因为本地已经装了sbt, 修改下自己的sbt的path, 以及添加 idea 的env 就是上面的 `SBT_OPTS="-Dsbt.override.build.repos=true` 不然在idea里面 他又会走默认的地址了





#### reference

http://stackoverflow.com/questions/3868863/how-to-specify-jvm-maximum-heap-size-xmx-for-running-an-application-with-run

http://www.scala-sbt.org/release/docs/Proxy-Repositories.html


http://stackoverflow.com/questions/22794149/scala-play-sbt-change-order-of-resolvers

http://www.plotprojects.com/create-an-intellij-idea-project-with-library-sources-attached/

