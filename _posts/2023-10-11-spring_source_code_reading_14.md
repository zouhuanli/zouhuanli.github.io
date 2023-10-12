---
layout: post
title: "Spring源码阅读十四:DispatcherServlet的初始化流程"
date: 2023-10-12
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十四篇文章，本文主要简单介绍一下Spring MVC的概念，重点介绍DispatcherServlet的初始化过程。<br>
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、SpringMVC基础

## 1.什么是SpringMVC?<br>

Spring MVC是Spring提供的轻量级的MVC框架，M-模型model，V-视图view，C-控制器controller。SpringMVC是基于servlet的轻量级的MVC框架，主要用于简化Java
Web应用的开发。<br>
SpringMVC对应的组件分别是：Model主要指Service和DAO，主要功能是数据操作。<br>
Controller主要指DispatcherServlet和controller,主要功能是流程控制。<br>
View主要指JSP、FreeMaker等视图技术，主要功能是视图展示。<br>
SpringMVC的源码模块主要是spring-webmvc,和spring-web。spring-web提供了处理web请求的更高层次的抽象，由Servlet栈的spring-webmvc和Reactive栈的spring-webflux引用。<br>
spring-webmvc的一个核心类就是DispatcherServlet，通常翻译为前端控制器。DispatcherServlet实现了servlet接口，servlet接口的service接口最终在DispatcherServlet的doDispatch方法。

## 2.DispatcherServlet和ApplicationContext

DispatcherServlet是核心的前端控制器，主要用于Http请求分派。其类关系图如下:

![DispatcherServlet](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-12-spring_source_code_reading_14/DispatcherServlet.png)

可以看到其顶层接口是Aware和Servlet。<br>

DispatcherServlet和ApplicationContext我们可以看一下官网的这个图片。

![mvc-context-hierarchy](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-12-spring_source_code_reading_14/mvc-context-hierarchy.png)

可以看到DispatcherServlet的Context主要是初始化web相关的bean，如Controller、Resolver、handleMapping等。<br>
而RootWebApplicationContext主要是初始化基础Bean，如Service、DAO等，以及其他非web层的组件。


# 二、DispatcherServlet的初始化流程

DispatcherServlet的初始化流程是此篇文章的主要内容。<br>
我们从Servlet的init方法作为入口开始研读源码。
    todo~

# 三、SpringMVC的请求流程

SpringMVC的请求流程主要就是DispatcherServlet的doDispatch方法。<br>
这里引用开涛老师的一张图，如下：

![spring-springframework-mvc](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-12-spring_source_code_reading_14/spring-springframework-mvc.png)

主要流程如下：

1. 客户端(前端)发送请求到DispatcherServlet，DispatcherServlet请求handlerMapping，handlerMapping返回HandlerExecutionChain(包含handler/controller，和拦截器列表)。<br>
2. DispatcherServlet请求HandlerAdapter，HandlerAdapter请求handler/Controller。handler/Controller执行请求的方法，返回ModelAndView，或者不返回View只返回数据给response。<br>
3. DispatcherServlet请求ViewResolver，进行视图解析，得到View对象。<br>
4. DispatcherServlet将Model数据应用于View对象，进行视图渲染，返回前端数据。<br>

笔者将在下一篇文章结合DispatcherServlet源码详细介绍请求流程。
# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)<br>
4. http://jinnianshilongnian.iteye.com/