---
layout: post
title: "Spring源码阅读二:Spring IoC基础"
date: 2023-09-21
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第二篇文章，本文将简单介绍一下Bean的注册方式和注入依赖的方式，以及对IoC理念的理解。重点要理解IoC(DI)的思想。

# 一、如何理解IoC或DI

我们知道一个Bean也就是Spring IoC容器托管的一个实例，bean是对象，class信息或者注解配置或者xml配置是生成bean的定义(define)
信息，有一种类似
"元信息"的味道。那么如何理解IoC(控制反转)或DI(依赖注入)呢？
我们考察一种最简单的情况,如下：
假设我有类Beta，依赖类Sigma。

    public class Beta {
    private Sigma sigma;

     public void invokeSigmaMethod() {
        sigma.doSomething();
        }
     public void setSigma(Sigma sigma) {
    this.sigma = sigma;
    }

    public class Sigma {

    /**
     * an empty method
     */
    public void doSomething() {

        }
    }

我们要运行Beta的invokeSigmaMethod方法，必然需要实例话Sigma得到一个实例，如下：

    Beta beta = new Beta();
    Sigma sigma = new Sigma();
    beta.setSigma(sigma);
    beta.invokeSigmaMethod();

我们这样手动创建依赖对象(beta依赖sigma),并将客户端对象(beta)依赖的对象sigma设置(注入)到客户端中，这种操作就是手动注入手动管理依赖对象。这种
方式在大型项目依赖对象众多、依赖关系复杂很显然极具难度，极其繁琐。
那么我们希望一个框架或者容器或者什么第三方的架构，能够自动创建依赖对象，对客户端对象注入依赖对象，并且管理依赖对象的整个生命周期，这就是Spring
IoC容器所实现的。
那么我们看看Spring IoC容器是如何操作的。依旧是这两个类，如下：

    @Component
    public class Beta {
    // @Autowired
        private Sigma sigma;

        public void invokeSigmaMethod() {
        sigma.doSomething();
        }

        @Autowired
        public void setSigma(Sigma sigma) {
        this.sigma = sigma;
        }
    }

和

    @Component
    public class Sigma {

    /**
     * an empty method
     */
    public void doSomething() {

     }
    }

我们使用@Component注解将Beta和Sigma注册为Spring容器托管的Bean，默认为单例对象，其name名字为类型名字首字母小写。我们来看下使用方式。

    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
        Beta beta = applicationContext.getBean(Beta.class);
    beta.invokeSigmaMethod();
    }

我们看到我们根本没有手动创建Sigma对象，Spring容器自动创建了，并且Beta对象依赖的Sigma对象也已经设值了，已经注入了客户端依赖的对象。

那么如何比较理论的理解IoC？
IoC控制反转是避免开发者手动创建依赖对象和维护依赖关系，导致类与类之间高耦合，难于修改和调试。而是把创建依赖对象和维护依赖关系交给容器，有其维护
对象依赖关系以及依赖对象的生命周期，保证类与类之间松散耦合，易于集成和测试，整体结构灵魂。IoC和DI有什么区别？不严谨的讲，是一样的。硬要说的话，
笔者的理解是IoC是顶层设计思想，一种指导理念，DI则是其指导理念下的一个具体产物或者具体实现。

参考下图和这篇文章:
《Inversion of Control Containers and the Dependency Injection pattern》：[https://www.martinfowler.com/articles/injection.html]
Spring官网的图片：
![container](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_2/container-magic.png)

# 二、Bean的注册方式

这里只简单介绍一下Bean的注册的三种的配置方式

## 1.XML配置

如下使用使用bean标签，以及可以从其他xml文件的bean注册信息通过import导入。如下，对象sigma是通过component注解注册的所以xml会报红。好处是使
用简单，运用任何场景，坏处是配置繁琐，现在都是尽量少使用XML配置。

    <!--创建一个和@Component同名的Bean-->
    <bean name="beta" class="com.homura.bean.Beta">
        <constructor-arg  index="0" ref="sigma"/>
    </bean>
    <!--创建一个和@Component不同名的Bean-->
    <bean name="beta1" class="com.homura.bean.Beta">
        <constructor-arg index="0" ref="sigma"/>
    </bean>

## 2.JavaConfig配置

如下，使用@Configuration和@Bean注册bean。好处是使用简单，不需要配置XML，坏处是某些复杂的bean写法很困难。

    @Configuration
    public class MyBeanConfig {

     /**
     * 设置bean的name，如果不设置默认是方法名字
     * If left unspecified, the name of the bean is the name of the annotated method.
     *
     * @param sigma
     * @return
     */
    @Bean(name = "myBetaBean")
    public Beta myBetaBean(@Autowired Sigma sigma) {
        return new Beta(sigma);
        }
    }

## 3.注解配置

使用@Component，@Service，@Repository，@Controller等注解。指定Spring扫描注解的basePackage，包下面注解注册的Bean都能被Spring识别。
好处是使用简单，但是对一些无法修改源码的第三方jar的类无法使用，只有使用XML或者JavaConfig。有一点是@Configuration的类自身会注册为一个bean。
例如：

    @Component
    public class Sigma {

    /**
     * an empty method
     */
    public void doSomething() {
        System.out.println("Sigma.doSomething()");
        }
    }

三种方式没有绝对的好坏，从boot看趋势是注解更主流，尽量减少XML配置。

# 三、Bean依赖的注册方式

在第二点其实已经涉及到了，依赖注入主要有三种：构造器注入，setter注入，自动注入(注解注入)

## 1.构造器注入

通过设置构造器参数来完成依赖注入，如下：

    <bean name="beta" class="com.homura.bean.Beta">
        <constructor-arg  index="0" ref="sigma"/>
        </bean>

## 2.setter注入

通过设置属性(字段、域)来完成依赖注入：

    <bean name="beta2" class="com.homura.bean.Beta">
    <property name="sigma" ref="sigma"/>
    </bean>

## 3.自动注入(注解注入)

autowired注解可以修饰字段，构造器参数，setter方法。autowired默认是按type查找依赖bean注入的，若同一个type有多个符合条件的bean，需要搭配
@Qualifier一起使用。示例如下：

    @Service
    public class UserServiceImpl {
    @Autowired
    private UserMapper userDao;

    /**
     * 查找用户列表
     *
     * @return
     */
    public List<User> findUserList() throws Exception {
        return userDao.findUserList();
     }
    }

# 四、一些注意点
## 1.循环依赖

这是一个有趣的问题，很多文章讲解过，笔者会在后面专门开一篇文章探讨循环依赖问题。

## 2.同名Bean的覆盖问题

Component扫描的bean的name默认是类名字首字母小写，如果你有同名类，或者xml注册同样name的bean，那么后面扫描/注册的beanDefinition会覆盖前者
的，对应创建的singletonObject也会使用后置的。其源码位置为：
![container](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_2/beanDefinitionOverriding.png)

## 3.@Autowired和@Resource以及@Inject

@Autowired是spring的注解,作用于ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, Elem
entType.ANNOTATION_TYPE。
默认注入是byType的，可以搭配@Qualifier指定bean的name,可以搭配@Primary指定首选的依赖对象。
@Resource是javax定义的注解，可以作用于ElementType.TYPE, ElementType.FIELD, ElementType.METHOD。其默认按name注入依赖对象。
@Inject是javax.inject引入的，作用于ElementType.CONSTRUCTOR、ElementType.METHOD、ElementType.FIELD，用法@Autowired基本一致，可以
搭配@Named来注入指定name的依赖对象。

# 五、参考材料

1.Spring官方文档：[https://spring.io/projects/spring-framework](https://spring.io/projects/spring-framework)   <br>
2.《spring源码深度解析》(郝佳)    <br>
3.Inversion of Control Containers and the Dependency Injection
pattern：[https://www.martinfowler.com/articles/injection.html]