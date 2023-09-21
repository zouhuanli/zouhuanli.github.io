---
layout: post
title: "Spring源码阅读一:Spring框架组成"
date: 2023-09-21
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第一篇文章，旨在介绍Spring Framework的来源和Spring框架的各个module源码的主要功能。

# 一、Spring的起源

Spring早期就是为了简化JavaEE开发而出现，作为轻量级的JavaWEB开发框架。Spring在Ioc和AOP之上构建了各模块应用，适用
不同的场景，并衍生出SpringBoot和Spring Cloud 两个大开发框架。<br>
整体来说，Spring是开发Java后端应用的实质性标准。在Spring中，IoC是Spring最核心的基础，IoC解决了开发人员手动处理依
赖对象的问题，对象的依赖关系有Spring容器托管。在讨论基础的Spring框架（Spring framework）我们通常讨论的是Spring
基础的Ioc、AOP、MVC、事务等模块，也是"Spring源码阅读"这个系列所探讨的内容。Spring已经超越了开发框架的范畴，而是
演化成了一种Java开发生态。所以讨论Spring不能仅仅看成是一个开发框架。笔者阅读Spring源码的版本为6.0.11+,servlet
版本为6.0.0，jdk版本为jdk17，阅读工具为"IntelliJ IDEA 2023.2.2 (UltimateEdition)"。不同版本的Spring源码
会有不同，但是其核心设计思想和核心源码整体是相同，不会有太多的差异。环境需要注意一点就是笔者使用了"Tomcat 10.1.11"，
因此servlet的命名空间要由javax更新为jakarta。<br>
Spring源码地址为：https://github.com/spring-projects/spring-framework.git.<br>
笔者演示用源码地址为：https://github.com/zouhuanli/SpringMvcDemo.git.<br>

# 二、Spring组件总览

首先看下Spring官网的组件构成图：

![Spring-overview](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-21-spring_source_code_reading_1/spring-overview.png)

可以看到主要分为核心容器(IoC)、AOP、WEB、数据访问/集成(消息、事务、数据库)、TEST(测试集成)和其他等模块。

## 1.核心容器

核心容器模块包含：spring-core, spring-beans, spring-context, spring-context-support, 和spring-expression (SpringExpres
sion Language) 模块。<br>
spring-core:主要提供资源访问、类型转换、核心工具类等，供其他模块使用。从源码结构看提供aot提前编译,asm、cglib、objenesis生成AOP的字节
码的相关支持。<br>
spring-beans:Spring框架提供容器托管对象能力的核心模块，提供IoC(DI)能力。一个非常核心的类就是BeanFactory。<br>
spring-context:在core和beans之上，拓展了BeanFactory，提供了资源绑定、国际化、容器生命周期、事件发布等功能。<br>
ApplicationContext，应用上下文是其中的核心类。ApplicationContext一般也是客户端启动Spring容器的类，客户端 <br>
不会直接操纵BeanFactory。

![spring-core和spring-beans](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-21-spring_source_code_reading_1/core_and_beans.png)

spring-expression：提供spring表达式。

## 2.Data Access/Integration（数据访问／集成）

spring-jdbc:提供了JDBC的集成
spring-tx:事务模块，支持编程和声明式事务管理
spring-oxm:支持Object/XML映射，将Java对象映射成XML数据或者将XML数据映射成Java对象。
spring-orm：对象-关系映射的API，支持Spring事务管理。
spring-jms:Java消息服务。

## 3.Web模块

Spring的Web模块包含Web,webmvc,websocket,webflux。
spring-web:基本的web功能，如WebApplicationContext应用上下文，multipart文件上传功能等
spring-websocket:websocket服务。
spring-webmvc:Spring MVC Web框架。
spring-webflux：响应式异步非阻塞的Web框架。
portlet模块已经移除。

![spring-web](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-21-spring_source_code_reading_1/web.png)

## 4.AOP

spring-aop:提供AOP支持，AOP通常使用在日志、权限、性能、监控等与业务代码横切的场景中，避免与业务代码耦合。
spring-aspects:提供和AspectJ的集成，AspectJ是一个成熟的面向切面编程(AOP)框架。

## 5.其他模块

spring-instrumentation:类工具和类加载器相关。
spring-messaging：消息模块。
spring-jcl:JCL日志模块。
spring-test:提供单元测试支持，如Junit。

# 三、Spring学习材料

有非常非常多学习Spring的书籍、文章、视频，笔者建议先把官网文档学习和研究一遍。地址如下：
SpringFramework文档：https://spring.io/projects/spring-framework

![springdoc](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-21-spring_source_code_reading_1/springdoc.png)

以及归档文件：https://docs.spring.io/spring-framework/docs/
可以下载离线文档。以及Spring官方的GitHub的源码地址：https://github.com/spring-projects/spring-framework.git.

# 四、参考材料

1.Spring官方文档：https://spring.io/projects/spring-framework    <br>
2.《spring源码深度解析》(郝佳)    <br>