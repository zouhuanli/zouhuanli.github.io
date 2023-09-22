---
layout: post
title: "Spring源码阅读三:Spring IoC容器整体结构"
date: 2023-09-22
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第三篇文章，本文将重点介绍BeanFactory和ApplicationContext两个顶层类的设计。
为什么先看两个类开始？
我们阅读官方文档有这样一个描述"the BeanFactory provides the configuration framework and basic functionality, and the
ApplicationContext adds more enterprise-specific functionality"。BeanFactory提供了Spring Ioc的配置获取和基本功能，
ApplicationContext拓展了BeanFactory并添加了更多企业级的功能，如资源获取、事件发布、国际化等。ApplicationContext也是客户端创建、
启动、使用容器的基本对象。

如何阅读源码？阅读源码其实一种非常枯燥的工作，而且很容易沉迷细节从而只见树木不见森林。而且现在成熟框架的源码随着各种更新，代码量已经非常
大了，不可能全部看完。下面分享一下笔者的阅读思路：

1. 自顶向下和自底向上的思想相结合。自顶向下自然是从框架的整体设计，整体结构，最高层抽象入手，逐层往下不断具体化，这样有助于把握框架的整体
   避免过早迷失在细节之中。自底向上是从具体一个功能、方法入手，不断深入，不断探索执行流程的高层级抽象，这样有助于理解具体细节和具体功能的执
   行流程

2. 从官方文档、经典书籍入手。先不要自己随意就阅读源码，参考官方文档、经典书籍的阅读思路和方法。

3. 实用着手。技术毕竟不是纯粹科学研究，是要落地到实际生产生活中解决实际问题的。可以从一些具体问题入手，阅读源码去探究这些问题的解决方式。
   可以跟踪(DEBUG)一些核心的方法、类的执行流程加深理解，也可以实际去运行官方的test类，像Spring的源码的Test是非常齐全的。

下面本篇文章将从整体上去探讨Spring容器的核心的几个顶层类：BeanFactory、BeanRegistry,BeanDefinition,ApplicationContext,旨在整体
把握Spring IoC容器的结构。

# 一、BeanFactory和BeanRegistry

BeanFactory是Spring IoC容器最核心的类，顾名思义BeanFactory是使用工厂模式创建的用于创建、管理、销毁Bean的工厂。创建Bean需要读取定义
Bean的配置，如读取xml的<bean>标签配置的bean，或者使用@Component等注解定义的Bean，因此BeanFactory还应存储Bean定义信息BeanDefinit
ion。为什么不直接跳过存储和管理BeanDefinition直接创建bean实例？笔者的理解是：首先，BeanDefinition是一种元信息，每次创建Bean都要读取其从而创建Bean
,自然需要保存：其次，Spring容器不是只有单例，也不是启动/刷新容器才创建Bean，所以需要保存BeanDefinition供创建Bean时候使用。
先看下BeanFactory的类关系图，来自IDEA自带的。如下：

![DefaultListableBeanFactory](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/DefaultListableBeanFactory.png)

## 1.BeanFactory

BeanFactory顶层接口如下：

![BeanFactory](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/BeanFactory.png)

可以看出主要从容器获取Bean、判断Bean是否存在、获取Bean的类型和别名、Bean是否是单例等方法。
从上面的类关系图可以看到,BeanFactory的子接口有:
ListableBeanFactory:在BeanFatory增加了Bean定义信息的计数，获取类型的BeanName列表等可迭代获取容器里面的Bean的信息。
HierarchicalBeanFactory：主要设置父容器，以及判断从当前容器而不是父容器判断Bean是否存在。
ConfigurableBeanFactory：可参数化/配置的容器,可以设置类加载器、属性编辑器、BeanPostProcessor(bean后置处理器)、销毁Bean等。
AutowireCapableBeanFactory：提供自动注入/装配的容器。
ConfigurableListableBeanFactory：ListableBeanFactory和ConfigurableBeanFactory的组合。
可以看到越底层的接口功能越细化，不断拓展BeanFactory的功能，这也是使用继承的目的。
BeanFactory的抽象类层次是：AbstractBeanFactory ---> AbstractAutowireCapableBeanFactory --->DefaultListableBeanFactory。
为什么选择DefaultListableBeanFactory？因为ApplicationContext持有的BeanFactory对象就是DefaultListableBeanFactory类型的。
现在我们关注到三个顶层的接口BeanFactory，BeanDefinitionRegistry，SingletonBeanRegistry。
BeanFactory是容器的基本功能，主要提供注册Bean、管理Bean、获取Bean和销毁Bean等和Bean对象相关的功能。
BeanDefinitionRegistry是Bean定义信息的注册中心， 指的是将<bean>
标签、@Component、@ComponentScan等注解定义的类信息存储，以供BeanFactory创建Bean使用。

	/** Map of bean definition objects, keyed by bean name. */
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

	/** Map from bean name to merged BeanDefinitionHolder. */
	private final Map<String, BeanDefinitionHolder> mergedBeanDefinitionHolders = new ConcurrentHashMap<>(256);

我们看到DefaultListableBeanFactory的成员属性，这些beanDefinitionMap就是注册、存储BeanDefinition的地方。
SingletonBeanRegistry则很好理解了，单例的注册中心，DefaultSingletonBeanRegistry中保存注册单例的集合如下：

	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

	/** Set of registered singletons, containing the bean names in registration order. */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

这里使用了三级缓存singletonObjects，singletonFactories，earlySingletonObjects，其中singletonObjects是最终完全容器创建好之后的单例
注册中心。
为什么只注册单例？Spring默认只托管单例的生命周期，原型和scope的是创建好了就交给客户端类使用了，根本不保存到singletonObjects。

# 二、BeanDefinition

BeanDefinition指的是Bean的定义信息，是类型信息，不是一个具体的Bean实例。

## 1.BeanDefinition

BeanDefinition设计的类关系图如下：

![BeanDefinition](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/BeanDefinition.png)

可以看到除了基本的RootBeanDefinition，还有关于注解、扫描、配置的bean定义。具体区别阅读源码时候应该可以看到。
BeanDefinition有class、scope、lazyInit、primary等诸多属性，和XML的定义的标签对应的。

	@Nullable
	private String scope = SCOPE_DEFAULT;

	private boolean abstractFlag = false;

	@Nullable
	private Boolean lazyInit;

	private int autowireMode = AUTOWIRE_NO;

	private int dependencyCheck = DEPENDENCY_CHECK_NONE;

	@Nullable
	private String[] dependsOn;

	private boolean autowireCandidate = true;

	private boolean primary = false;

	private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

	@Nullable
	private Supplier<?> instanceSupplier;

	private boolean nonPublicAccessAllowed = true;

	private boolean lenientConstructorResolution = true;

	@Nullable
	private String factoryBeanName;

	@Nullable
	private String factoryMethodName;

## 2.BeanDefinitionHolder

BeanDefinitionHolder是BeanDefinition的包装类，具体如下：

    public class BeanDefinitionHolder implements BeanMetadataElement {

	private final BeanDefinition beanDefinition;

	private final String beanName;

	@Nullable
	private final String[] aliases;

    //省略其他
    }

## 3.BeanDefinitionReader

BeanDefinitionReader是读取Bean定义信息的读取器，涉及的类如下：

![DefinitionReader](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/DefinitionReader.png)

可以看到主要也分为XML配置和注解配置。

# 三、ApplicationContext

ApplicationContext翻译为应用上下文，拓展BeanFactory，增加了企业级的功能如资源加载、i18n消息国际化、事件发布、生命周期信息等。
ApplicationContext上层的类型/接口如下,

![ApplicationContext](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/ApplicationContext.png)

可以看到其继承了BeanFactory并拓展了企业级功能。
我们再看下ApplicationContext的具体实现类：

![ClassPathXmlApplicationContext](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-22-spring_source_code_reading_3/ClassPathXmlApplicationContext.png)

从图中我们看到ApplicationContext的具体实现类还是非常多的，不妨自顶向下做一个总的划分。
AbstractRefreshableApplicationContext：可以刷新配置的上下文，如XML配置的。
GenericApplicationContext：不刷新配置的上下文，主要是基于注册配置的。
再往下看，又按照配置的加载源来分：如从XML文件加载则是XML名字的，注解的则是注解的，groovy的则是groovy的ApplicationContext。
然后按应用类型可以分为简单ApplicationContext和WebApplicationContext。
限于篇幅，ClassPathXmlApplicationContext和AnnotationConfigApplicationContext是本系列的研究重点的类型。
ApplicationContext的简单使用，定义好Bean信息（XML或/和注解），然后初始化容器，使用ApplicationContext。如：

    public static void main(String[] args) {
            ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
            applicationContext.start();
            Beta beta = applicationContext.getBean("beta",Beta.class);
            beta.invokeSigmaMethod();
       }



# 四、参考材料

1.Spring源码(版本6.0.11)
2.《spring源码深度解析》(郝佳)    <br>
