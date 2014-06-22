---
layout: post
title: "开始用jetty调代码"
description: ""
category: "work"
tags: [java,tomcat,jetty,web，eclipse]
---


今天我研究了一上午，尝试了下用jetty配合eclipse调试web项目。

####why choose jetty
eclipse本来不就有server插件的嘛，可以直接用tomcat呀，为嘛要用jetty呀？

嗯哼，前几天调试代码的时候，有不知道那个文件被锁住还是神马的，class没有拷贝到eclipse的temp tomcat目录里面。害得我又浪费时间，以为是小伙伴的代码出了问题，各种看别人代码。。。

####why tomcat with eclipse sometimes make me crazy
eclipse的workplace里面项目多了以后好像是经常性抽筋额。是神马原因造成的呢？

嗯哼，我就说说maven的web项目，在eclispe里面每次按下`ctrl+s`保存代码时maven会把编译好的代码放到target里面，然后如果配置了server的话，还会编译一份到eclipse的workplace里面的某号temp server里面比如
	
`X:\project_workplace\.metadata\.plugins
\org.eclipse.wst.server.core\tmpx\m2ewebapp\WEB-INF\classes`
	
嗯，一份代码，竟然需要复制到2个地方。我猜可能是内存不够或者某原因文件被锁。so，总之没复制全，导致xxclass not found，或者别的类似错误。

那如果我用jetty的话，会怎样？我只要指定我的webapp的path是target目录，然后就不用管他了，也就没有了所谓的第二次的复制，异常发生机率更小了，即使发生异常，也只要delete掉target，然后rebuild就行了，不像eclipse还要clean server，有时clean server还会报exception，真让人想掀桌！！！

####config jetty plugin in maven
原来如此，那就赶紧配置maven的eclipse插件吧

1.在pom里面的 `<build>` 下加一个plugin


```xml
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
```

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



- - - -
6月22号更新

最近切换到idea后，尝试下新的玩法。

总的配置文件

```xml
	<build>
    <!-- 配置待编译的源码和配置文件 -->
        <sourceDirectory>src/main/java</sourceDirectory>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        <resources>
            <resource>
                <directory>src/main/resource</directory>
            </resource>
            <resource>
                <directory>src/main/conf/${envcfg.dir}</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>src/test/resource</directory>
            </testResource>
        </testResources>
        <directory>${project.basedir}/target</directory>
        <finalName>${project.artifactId}</finalName>
        <outputDirectory>${project.build.directory}/${project.build.finalName}/WEB-INF/classes</outputDirectory>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-war-plugin</artifactId>
				<version>2.1.1</version>
				<configuration>
					<webResources>
						<resource>
							<directory>WebContent</directory>
						</resource>
					</webResources>
				</configuration>
			</plugin>
            <plugin>
                <groupId>org.mortbay.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <configuration>
                    <stopKey>foo</stopKey>
                    <stopPort>9998</stopPort>
                    <webAppSourceDirectory>${basedir}/WebContent</webAppSourceDirectory>
                    <webApp>
                        <contextPath>/</contextPath>
                        <descriptor>${basedir}WebContent/WEB-INF/web.xml</descriptor>
                    </webApp>
                    <!-- you use default when develop -->
                    <!--<connectors>-->
                        <!--<connector-->
                                <!--implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">-->
                            <!--<port>8081</port>-->
                            <!--<maxIdleTime>60000</maxIdleTime>-->
                        <!--</connector>-->
                    <!--</connectors>-->
                    <!-- 自动扫描用于热启动 -->
                    <scanIntervalSeconds>3</scanIntervalSeconds>
                </configuration>
                <executions>
                    <execution>
                        <id>start-jetty</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>run</goal>
                        </goals>
                        <configuration>
                            <daemon>true</daemon>
                            <reload>automatic</reload>
                        </configuration>
                    </execution>
                    <execution>
                        <id>stop-jetty</id>
                        <phase>post-integration-test</phase>
                        <goals>
                            <goal>stop</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
		</plugins>
	</build>
```
 
直接终端下 mvn jetty:run 启动，如果加上启动参数便可以远程调试，是不是很像play

不过我还是直接idea里直接跑吧。。。

