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
2.Tomcat启动流程：启动流程、类加载<br>
3.服务器Server<br>
4.服务Service<br>
5.连接器Connector和线程池<br>
6.容器Container：Host、Engine、Context、Wrapper<br>
7.Http请求处理流程上:调试追踪一个HTTP请求的处理过程<br>
8.http请求处理流程下：调试追踪一个HTTP请求的处理过程<br>

本系列的源码工程为：[HowTomcatWorksSourceCode](https://github.com/zouhuanli/HowTomcatWorksSourceCode.git)。


TODO~~~

# 一、简单使用

# 二、Web和Servlet

# 三、参考材料

# 四、整体架构

# 五、核心流程

# 六、参考材料

1. https://tomcat.apache.org/tomcat-10.1-doc/index.html  <br>
2. 《深入剖析Tomcat》 <br>

