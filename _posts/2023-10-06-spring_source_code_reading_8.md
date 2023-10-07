---
layout: post
title: "Spring源码阅读八:AOP基础"
date: 2023-10-06
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第八篇文章，本篇文章将简单介绍一下Spring AOP的使用。<br>
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、基础

IoC提供自动管理Bean和其依赖关系的框架，将开发者从对象实例的依赖关系管理解放出来。<br>
OOP是对现实世界的事物的分类抽象，将事务分类，然后关注每一类的动态行为和静态属性。AOP提供了一种不同与OOP的编程思路,<br>
即将通用的、和业务无关或有关的逻辑，横插到相关的业务代码中，而不需要修改现有的业务代码，减少代码耦合，主要用于通用逻辑行为插入、安全检查、日志、事务、监控等场景。
下面是一些AOP相关的基础术语：<br>

1. 连接点Joinpoint：在什么地方执行aop插入的代码，在Spring中可以简单理解为一个方法。<br>
2. 切入点Pointcut：连接点的集合，在Spring中简单理解为某些方法的集合。<br>
3. 通知Advice：AOP要插入的代码，要执行的逻辑代码，简单理解在连接点（方法)前后要做什么。<br>
4. 切面Aspect：通知和切入点的组合，比如定义一个切面，里面包含切入点也就是一些方法的组合，以及定义为通知（如前置通知）的代码等，统一为一个类。
5. 目标对象：被通知的代码处理的原始对象，如切入点是方法aMethod，类A包含该方法，则该切面可以作用有类A，类A的对象(假定是单例)
   称为原始对象。<br>
6. 代理对象：Spring AOP为了将切面的增强处理作用于目标对象，而创建的代理对象。分为JDK动态代理和CGLIB动态代理。<br>

五种通知类型：

1. 前置通知：@Before注解标注的通知，在连接点执行之前的通知。
2. 后置通知；@AfterReturning注解标注的通知,在连接点(方法)正常完成之后的执行的通知，可以获取原方法的返回结果。
3. 最终通知：@After注解标注的通知，在连接点退出之后执行的通知，无论异常退出还是正常退出，主要为了做一下最终操作如释放资源，有点类似finally。PS：所以为什么这个不叫后置通知，把最终通知改成@Final之类的？
4. 环绕通知：@Around注解标注的通知，在一个连接点(方法)前后执行的通知。
5. 异常通知：@AfterThrowing，在连接点抛出异常执行的通知。

基本的AOP概念介绍到这里，下面使用XML和注解的两种形式简单使用案例介绍Spring AOP的使用方法。

# 二、XML方式配置

先定义切面类
```java
package com.homura.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Component;

/**
 * 简单的日志切面，主要是演示五种通知的用法
 *
 * @author zouhl
 */
//@EnableAspectJAutoProxy
//@Component
//@Aspect
public class LogAspect {
   private static final Logger LOGGER = LoggerFactory.getLogger(LogAspect.class);

   /**
    * 定义切入点
    */
   // @Pointcut("execution (* com.homura.controller.*.*(..))")
   private void pointCutMethod() {
   }


   /**
    * 环绕通知
    *
    * @param joinPoint
    * @return
    * @throws Throwable
    */
   // @Around("pointCutMethod()")
   public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
      LOGGER.info("环绕通知:进入方法");
      Object originalResult = joinPoint.proceed();
      LOGGER.info("环绕通知:退出方法");
      return originalResult;
   }

   /**
    * 前置通知
    */
   // @Before("pointCutMethod()")
   public void doBefore() {
      LOGGER.info("前置通知");
   }

   /**
    * 最终通知
    */
   // @After("pointCutMethod()")
   public void doAfter() {
      LOGGER.info("最终通知");
   }

   /**
    * 后置通知
    *
    * @param result
    */
   // @AfterReturning(value = "pointCutMethod()", returning = "result")
   public void doAfterReturning(Object result) {
      LOGGER.info("后置通知,返回值：" + result);
   }

   /**
    * 异常通知
    *
    * @param e
    */
   // @AfterThrowing(value = "pointCutMethod()", throwing = "e")
   public void doAfterThrowing(Exception e) {
      LOGGER.info("异常通知,异常值：" + e);
   }

}


```
使用XML配置Spring AOP
```xml

    <!--AOP配置-->
    <bean id="aspect" class="com.homura.aspect.LogAspect"/>
    <aop:aspectj-autoproxy/>
    <aop:config>
        <aop:aspect id="aspect" ref="aspect">
            <aop:pointcut id="pointcut" expression="execution (* com.homura.controller.*.*(..))"/>
            <aop:before method="doBefore" pointcut-ref="pointcut"/>
            <aop:after method="doAfter" pointcut-ref="pointcut"/>
            <aop:around method="doAround" pointcut-ref="pointcut"/>
            <aop:after-returning method="doAfterReturning" pointcut-ref="pointcut" returning="result"/>
            <aop:after-throwing method="doAfterThrowing" pointcut-ref="pointcut" throwing="e"/>
        </aop:aspect>
    </aop:config>
```
测试类
```java
package com.homura.bean;

import com.homura.controller.UserController;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AspectTest {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
        UserController userController = (UserController) applicationContext.getBean("userController");
        userController.list(null, null);


    }
}

```
测试结果如下：
```text
2023-10-06 22:56:24,610|INFO |          LogAspect.java:49  |main|前置通知
2023-10-06 22:56:24,611|INFO |          LogAspect.java:38  |main|环绕通知:进入方法
2023-10-06 22:56:25,271|INFO |          LogAspect.java:67  |main|后置通知,返回值：ModelAndView [view="userList"; model={dateTime=2023-10-06T22:56:24.612393500, userList=[User(uid=3, uname=bb, password=123, gender=null, phone=null, email=null, address=null), User(uid=7, uname=L, password=123, gender=男, phone=12345678900, email=123@qq.com, address=我家), User(uid=8, uname=123, password=213, gender=男, phone=321, email=213, address=3), User(uid=11, uname=, password=, gender=null, phone=null, email=null, address=null), User(uid=12, uname=111, password=222, gender=null, phone=null, email=null, address=null), User(uid=13, uname=333, password=333, gender=null, phone=null, email=null, address=null), User(uid=14, uname=666, password=666, gender=null, phone=null, email=null, address=null), User(uid=15, uname=777, password=777, gender=null, phone=null, email=null, address=null), User(uid=16, uname=16, password=16, gender=女, phone=1340, email=11, address=16道路), User(uid=17, uname=17, password=17, gender=男, phone=111, email=11, address=111), User(uid=18, uname=19, password=20, gender=222, phone=222, email=222, address=222)]}]
2023-10-06 22:56:25,271|INFO |          LogAspect.java:40  |main|环绕通知:退出方法
2023-10-06 22:56:25,271|INFO |          LogAspect.java:57  |main|最终通知

```
通知的顺序可以通知Order来指定，但是两个相同的通知在同一个连接点执行顺序是未知的。

# 三 、注解配置
切面类如下：
```java
package com.homura.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Component;

/**
 * 简单的日志切面，主要是演示五种通知的用法
 *
 * @author zouhl
 */
@EnableAspectJAutoProxy
@Component
@Aspect
public class LogAspect {
    private static final Logger LOGGER= LoggerFactory.getLogger(LogAspect.class);

    /**
     * 定义切入点
     */
    @Pointcut("execution (* com.homura.controller.*.*(..))")
    private void pointCutMethod() {
    }


    /**
     * 环绕通知
     *
     * @param joinPoint
     * @return
     * @throws Throwable
     */
    @Around("pointCutMethod()")
    public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
        LOGGER.info("环绕通知:进入方法");
        Object originalResult = joinPoint.proceed();
        LOGGER.info("环绕通知:退出方法");
        return originalResult;
    }

    /**
     * 前置通知
     */
    @Before("pointCutMethod()")
    public void doBefore() {
        LOGGER.info("前置通知");
    }

    /**
     * 最终通知
     */
    @After("pointCutMethod()")
    public void doAfter() {
        LOGGER.info("最终通知");
    }

    /**
     * 后置通知
     *
     * @param result
     */
    @AfterReturning(value = "pointCutMethod()", returning = "result")
    public void doAfterReturning(Object result) {
        LOGGER.info("后置通知,返回值：" + result);
    }

    /**
     * 异常通知
     *
     * @param e
     */
    @AfterThrowing(value = "pointCutMethod()", throwing = "e")
    public void doAfterThrowing(Exception e) {
        LOGGER.info("异常通知,异常值：" + e);
    }

}

```
测试主类：
```java
package com.homura.bean;

import com.homura.controller.UserController;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AspectTest {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
        UserController userController = (UserController) applicationContext.getBean("userController");
        userController.list(null, null);


    }
}

```
测试结果如下：
```text
2023-10-06 23:08:19,151|INFO |          LogAspect.java:38  |main|环绕通知:进入方法
2023-10-06 23:08:19,152|INFO |          LogAspect.java:49  |main|前置通知
2023-10-06 23:08:19,760|INFO |          LogAspect.java:67  |main|后置通知,返回值：ModelAndView [view="userList"; model={dateTime=2023-10-06T23:08:19.152436800, userList=[User(uid=3, uname=bb, password=123, gender=null, phone=null, email=null, address=null), User(uid=7, uname=L, password=123, gender=男, phone=12345678900, email=123@qq.com, address=我家), User(uid=8, uname=123, password=213, gender=男, phone=321, email=213, address=3), User(uid=11, uname=, password=, gender=null, phone=null, email=null, address=null), User(uid=12, uname=111, password=222, gender=null, phone=null, email=null, address=null), User(uid=13, uname=333, password=333, gender=null, phone=null, email=null, address=null), User(uid=14, uname=666, password=666, gender=null, phone=null, email=null, address=null), User(uid=15, uname=777, password=777, gender=null, phone=null, email=null, address=null), User(uid=16, uname=16, password=16, gender=女, phone=1340, email=11, address=16道路), User(uid=17, uname=17, password=17, gender=男, phone=111, email=11, address=111), User(uid=18, uname=19, password=20, gender=222, phone=222, email=222, address=222)]}]
2023-10-06 23:08:19,760|INFO |          LogAspect.java:57  |main|最终通知
2023-10-06 23:08:19,760|INFO |          LogAspect.java:40  |main|环绕通知:退出方法

```
本文至此简单介绍了一下SpringAOP的简单使用，为后续源码阅读做一下入门工作。


# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)
