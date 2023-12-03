---
layout: post
title: "Tomcat源码解读一:Tomcat整体架构"
date: 2023-11-23
tags: [ Tomcat ]
comments: true
author: zouhuanli
---

此系列是结合Tomcat使用来解读Tomcat源码的专栏，主要参数材料为Tomcat源码和《深入剖析Tomcat》这本书。《深入剖析Tomcat》这本书挺老了，讲的也是Tomcat4和5，但是对应Tomcat整体体系结构和内部实现原理讲的比较深入，仍然是必读的解读Tomcat的书籍。<br>
笔者不会完全按照这本书的目录解读，会做一些内容上的删减，比如"日志"、"安全性”、"部署器"、"Manager"、"JMX"等章节，读者若对其感兴趣请参数书籍和官网文档去了解。<br>
然后笔者在书籍选择会偏向选择美国的相关书籍，毕竟计算机行业是从美国发展起来的。<br>

本系列初步的目录如下：<br>
1.整体架构：简单入门、Web和Servlet、整体架构、核心流程<br>
2.Tomcat启动流程：启动流程、类加载、各级组件/容器初始化和启动<br>
3.服务器Server<br>
4.服务Service<br>
5.连接器Connector和线程池<br>
6.容器Container：Host、Engine、Context、Wrapper<br>
7.Http请求处理流程上:调试追踪一个HTTP请求的处理过程<br>
8.http请求处理流程下:调试追踪一个HTTP请求的处理过程<br>

本系列的源码工程为：[HowTomcatWorksSourceCode](https://github.com/zouhuanli/HowTomcatWorksSourceCode.git)。


# 一、简单使用

使用Tomcat可以通过单独的SpringMVC的Web项目启动，或者通过SpringBoot启动tomcat。<br>
单独使用主要是配置Servlet：
```xml

    <servlet>
        <servlet-name>springmvc-demo</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc-demo</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```
通过SpringBoot启动则主要是引入WEB(tomcat)的starter。
```xml
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring.boot.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

然后则可以通过浏览器或Http Client发送HTTP请求，这里不再赘述了。

# 二、Tomcat和Servlet

Tomcat是什么？Tomcat是Web服务器、Servlet容器。我们谈及Web服务器并不是特指的是HTTP Web Server，WEB服务器本身不是只支持HTTP协议请求的。

Tomcat自身是使用Java开发的，实现了Servlet标准。Servlet是Java处理Web请求的顶层API，接受和处理Web请求。

我们看下官方的注释：
```java
 * A servlet is a small Java program that runs within a Web server. Servlets receive and respond to requests from Web
 * clients, usually across HTTP, the HyperText Transfer Protocol.
```
HttpServlet是继承自Servlet，这表面Servet本身不是特指HTTP协议请求的处理API。

Wiki对Tomcat的说明是"Tomcat是实现Servlet、WebSocket标准的开源Java Web服务器，通常指的是处理HTTP协议的WEB服务器"。


# 三、整体架构

要分析Tomcat的整体架构可以从启动方法开始研究。

分析流程是：Bootstrap--->Catalina---->Server---->Service--->Engine-->Container---->Host--->Context--->Wrapper--->Servlet这个源码方向。

一个Tomcat启动的Server服务器只有一个，而有多个Service服务。executors线程池、connectors连接器(列表)也是由一个StandardService（Service）创建的。

Engine、Container、Host、Context、Wrapper都称为"容器",容器类的类继承关系图如下：

![Container](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-23-tomcat_source_code_reading_1/Container.png)

Connectors连接器是连接Service和Container的结构，Connectors会创建和启动protocolHandler协议处理器用以处理http、ajp、https协议。

Tomcat整体的架构，笔者在网上找到这个图总结的比较好的：

![tomcat-x-design.jpeg](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-23-tomcat_source_code_reading_1/tomcat-x-design.jpeg)


# 四、核心流程

## 4.1 初始化和启动流程

启动流程可以从Bootstrap启动类开始解读,一步一步来到Context，最后来到Wrapper和Servlet。
启动流程如下，都是先初始化和启动父容器再初始化和启动子容器。


![Bootstrap_start-0](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-23-tomcat_source_code_reading_1/Bootstrap_start-0.png)

需要注意这段代码：
```java
   // Start our child containers, if any
        Container[] children = findChildren();
        List<Future<Void>> results = new ArrayList<>(children.length);
        for (Container child : children) {
            results.add(startStopExecutor.submit(new StartChild(child)));
        }

        MultiThrowable multiThrowable = null;

        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Throwable e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                if (multiThrowable == null) {
                    multiThrowable = new MultiThrowable();
                }
                multiThrowable.add(e);
            }

        }
```
这里是启动线程池去启动子容器children，父容器在阻塞等待子容器的启动结果。


## 4.2 Http请求处理流程

我们之前分析DispatchServlet知道DispatchServlet实现了Servlet接口，并将service方法转发到dispatch方法。但DispatchServlet的创建、初始化、销毁都是Servlet容器(如Tomcat)来操作的。

Servlet#service方法也是由Tomcat调用的,上面启动流程中Wrapper将Servlet修饰为StandardWrapper。

```java

    /**
     * The (single) possibly uninitialized instance of this servlet.
     */
    protected volatile Servlet instance = null;
```

StandardWrapper是Servlet的修饰器，我们可以从Servlet的service方法解读引用流程。

Http请求的基本流程如下，后续笔者会跟踪源码流程给出Http请求处理的流程解读。

![process-0](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-23-tomcat_source_code_reading_1/process-0.png)

# 五、参考材料

1. https://tomcat.apache.org/tomcat-10.1-doc/index.html  <br>
2. 《深入剖析Tomcat》 <br>
3. 《HTTP权威指南》

