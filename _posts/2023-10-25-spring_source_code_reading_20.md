---
layout: post
title: "Spring源码阅读二十:BeanFactoryPostProcessor解读"
date: 2023-10-25
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第二十篇文章，本文简单解读一下BeanFactoryPostProcessor这个类。<br>
BeanFactoryPostProcessor是在BeanFactory创建和初始化之后执行的容器级别后置处理器，之前解读IoC容器初始化流程(以及AnnotationConfigApplicationContext)中
由这个invokeBeanFactoryPostProcessors方法，调用了beanFactory的后置处理器:

```java

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
```

本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

Todo~