---
layout: post
title: "Spring源码阅读六:Bean的创建过程和Bean的生命周期"
date: 2023-09-26
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第六篇文章，本篇文章主要介绍Bean的创建过程和Bean的生命周期。
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>
本文着重介绍Bean的创建过程doGetBean方法和Bean的生命周期，Bean的生命周期包括实例化(instantiation)、属性填充(
setter/populate)、初始
化(initialize)、使用中、销毁,以及围绕创建Bean的这个三个过程中的各种后置处理器BeanPostProcessor和Aware接口。<br>
本篇文章将从refresh方法到doGetBean方法，具体介绍Bean的创建过程和Bean的生命周期。

# 一、入口

我们先看入口的简单例子：

```java
public static void main(String[]args){
        ClassPathXmlApplicationContext applicationContext=new ClassPathXmlApplicationContext("springmvc-context.xml");
        //按类型已经无法获取了，因为有两个Beta类型的bean。Exception in thread "main" org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.homura.bean.Beta' available: expected single matching bean but found 2: beta,beta1
        // Beta beta = applicationContext.getBean(Beta.class);
        //  beta.invokeSigmaMethod();

        Beta beta=(Beta)applicationContext.getBean("beta");
        Beta beta1=(Beta)applicationContext.getBean("beta1");
        Beta beta2=(Beta)applicationContext.getBean("beta2");
        System.out.println(beta);
        System.out.println(beta1);
        System.out.println(beta2);

        }
```

我们依旧进入refresh方法，然后进入<strong>finishBeanFactoryInitialization</strong>方法，如下：

```java
    // Instantiate all remaining (non-lazy-init) singletons.实例化所有的非懒加载的Bean实例。
				finishBeanFactoryInitialization(beanFactory);
```

可以看到这里从之前中注册到容器的Bean配置beanDefinition实例化Bean，真正创建了Bean实例。我们继续往下看：

```java
    /**
 * Finish the initialization of this context's bean factory,
 * initializing all remaining singleton beans.
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory){
        // Initialize conversion service for this context.
        if(beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME)&&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME,ConversionService.class)){
        beanFactory.setConversionService(
        beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME,ConversionService.class));
                 }

        // Register a default embedded value resolver if no BeanFactoryPostProcessor
        // (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
        // at this point, primarily for resolution in annotation attribute values.
        if(!beanFactory.hasEmbeddedValueResolver()){
        beanFactory.addEmbeddedValueResolver(strVal->getEnvironment().resolvePlaceholders(strVal));
                            }

        // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
        String[]weaverAwareNames=beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class,false,false);
        for(String weaverAwareName:weaverAwareNames){
        getBean(weaverAwareName);
                 }

        // Stop using the temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(null);

        // Allow for caching all bean definition metadata, not expecting further changes.
        beanFactory.freezeConfiguration();

        // Instantiate all remaining (non-lazy-init) singletons.
        beanFactory.preInstantiateSingletons();
        }

```

我们看到首先创建了框架自己的一些Bean，然后创建其他(客户端定义，应用创建)的bean。我们继续跟踪执行流程，进入preInstantiateSingletons方法，如下：

![beanDefinitionNames](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-26-spring_source_code_reading_6/beanDefinitionNames.png)

像"org.springframework.aop.config.internalAutoProxyCreator"这些都是框架自己创建的Bean。<br>
图里的isFactoryBean方法判断了FactoryBean和Bean实例自己，FactoryBean是创建一个类型的Bean的工厂对象，通过getObject返回要生成的Bean。<br>
现在进入<strong>getBean</strong>方法，然后进入<strong>doGetBean</strong>方法，这个是创建Bean实例的方法。

# 二、bean创建流程

## 1.doGetBean
我们先看doGetBean方法，这个方法有点长，我做了一些删减和解读：

```java
protected<T> T doGetBean(
        String name,@Nullable Class<T> requiredType,@Nullable Object[]args,boolean typeCheckOnly)
        throws BeansException{

        String beanName=transformedBeanName(name);
        Object beanInstance;

        // Eagerly check singleton cache for manually registered singletons.提前返回从三级缓存获取beanName的bean
        Object sharedInstance=getSingleton(beanName);
        if(sharedInstance!=null&&args==null){
        if(logger.isTraceEnabled()){
        if(isSingletonCurrentlyInCreation(beanName)){
        //注意这行日志,返回beanName的Bean实例，这个实例尚未初始化完成，以应对循环依赖的场景。
        //也就是这三个缓存为解决循环依赖做了一些工作。
        logger.trace("Returning eagerly cached instance of singleton bean '"+beanName+
        "' that is not fully initialized yet - a consequence of a circular reference");
                }
        }
        //对bean实例做一些适配
        beanInstance=getObjectForBeanInstance(sharedInstance,name,beanName,null);
            }
        else{
        //从父容器获取
        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory=getParentBeanFactory();
        if(parentBeanFactory!=null&&!containsBeanDefinition(beanName)){
        // Not found -> check parent.
        String nameToLookup=originalBeanName(name);
        if(parentBeanFactory instanceof AbstractBeanFactory abf){
        return abf.doGetBean(nameToLookup,requiredType,args,typeCheckOnly);
            }
        else if(args!=null){
        // Delegation to parent with explicit args.
        return(T)parentBeanFactory.getBean(nameToLookup,args);
             }
        else if(requiredType!=null){
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup,requiredType);
             }
        else{
        return(T)parentBeanFactory.getBean(nameToLookup);
             }
        }
        //从当前容器获取
        //获得bean配置信息BeanDefinition
        RootBeanDefinition mbd=getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd,beanName,args);
        //注册dependsOn导入的Bean实例
        // Guarantee initialization of beans that the current bean depends on.
        String[]dependsOn=mbd.getDependsOn();
        for(String dep:dependsOn){
        registerDependentBean(dep,beanName);
        try{
        getBean(dep);
             }

                }
        // 创建单例，默认的单例创建方法入口
        // Create bean instance.
        if(mbd.isSingleton()){
        sharedInstance=getSingleton(beanName,()->{
        return createBean(beanName,mbd,args);
            });
        //对bean实例做一些适配
        beanInstance=getObjectForBeanInstance(sharedInstance,name,beanName,mbd);
            }
        //创建原型的Bean实例
        else if(mbd.isPrototype()){
        // It's a prototype -> create a new instance.
        Object prototypeInstance=null;
        beforePrototypeCreation(beanName);
        prototypeInstance=createBean(beanName,mbd,args);
        //对bean实例做一些适配
        beanInstance=getObjectForBeanInstance(prototypeInstance,name,beanName,mbd);
            }
        //创建scope的bean实例
        else{
        String scopeName=mbd.getScope();
        Scope scope=this.scopes.get(scopeName);
        Object scopedInstance=scope.get(beanName,()->{
        beforePrototypeCreation(beanName);
        try{
        return createBean(beanName,mbd,args);
            }
        finally{
        afterPrototypeCreation(beanName);
            }
        });
        //对bean实例做一些适配
        beanInstance=getObjectForBeanInstance(scopedInstance,name,beanName,mbd);
                }
             }
         }
        //对bean实例做一些适配
        return adaptBeanInstance(name,beanInstance,requiredType);
        }
```

我们看下这个方法:

```java
    @Nullable
protected Object getSingleton(String beanName,boolean allowEarlyReference){
        // Quick check for existing instance without full singleton lock
        Object singletonObject=this.singletonObjects.get(beanName);
        if(singletonObject==null&&isSingletonCurrentlyInCreation(beanName)){
            singletonObject=this.earlySingletonObjects.get(beanName);
        if(singletonObject==null&&allowEarlyReference){
            synchronized (this.singletonObjects){
                // Consistent creation of early reference within full singleton lock
                singletonObject=this.singletonObjects.get(beanName);
                    if(singletonObject==null){
                        singletonObject=this.earlySingletonObjects.get(beanName);
                             if(singletonObject==null){
                                ObjectFactory<?> singletonFactory=this.singletonFactories.get(beanName);
                                        if(singletonFactory!=null){
                                              singletonObject=singletonFactory.getObject();
                                                this.earlySingletonObjects.put(beanName,singletonObject);
                                                this.singletonFactories.remove(beanName);
                            }
                         }
                     }
                 }
             }
            }
        return singletonObject;
        }
```

注意这里，先从singletonObjects(一级缓存去获取)，获取不到从earlySingletonObjects去获取,再获取不到就singletonFactories获取，然后删除
singletonFactories的这个Bean实例，添加到earlySingletonObjects。这里返回singletonFactory提前暴露正在创建中的尚未完全初始化完成的Bean实例,
主要是为了解决循环依赖。
<br>

总的来说，doGetBean这里大致分为三步:<br>
1.从三级缓存获取。<br>
2.从父容器获取。<br>
3.从当前容器获取，按照scope的类型创建，默认XML配置和注册配置的Bean就是单例的，进入getSingleton方法创建单例。<br>
4.对返回的bean实例做一些适配。
## 2.getSingleton
我们继续进入getSingleton方法。我们先进入getSingleton方法，然后再读

```java
                 // Create bean instance.
				if (mbd.isSingleton()) {
                        sharedInstance = getSingleton(beanName, () -> {
                        try {
                        return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                            }
                         });
                        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                        }
```

这里的ctFactory<?> singletonFactory参数是一个ObjectFactory的函数式接口。

getSingleton方法如下：

```java

/**
 * Return the (raw) singleton object registered under the given name,
 * creating and registering a new one if none registered yet.
 * @param beanName the name of the bean
 * @param singletonFactory the ObjectFactory to lazily create the singleton
 * with, if necessary
 * @return the registered singleton object
 */
public Object getSingleton(String beanName,ObjectFactory<?> singletonFactory){
//获得singletonObjects的锁
synchronized (this.singletonObjects){
        //1.先从singletonObjects,这个最终的所有的单例缓存集合去获取
        //这是一个ConcurrentHashMap，存入最终的完全初始化完成的单例对象。
        Object singletonObject=this.singletonObjects.get(beanName);
        //不存在，开始创建新的单例
        if(singletonObject==null){
        //2.关注这行,加入beanName到singletonsCurrentlyInCreation这个Map
        beforeSingletonCreation(beanName);
        boolean newSingleton=false;
        try{
        //3.从ObjectFactory这个工厂类去创建单例，其只有一个getObject()方法，这个是一个工厂方法(factory-method).
        //这个类型类似于FactoryBean
        singletonObject=singletonFactory.getObject();
        newSingleton=true;
            }
        finally{
        //4.从singletonsCurrentlyInCreation移除beanName
        afterSingletonCreation(beanName);
            }
        if(newSingleton){
        //5.注意此方法，singletonObjects存入创建的Bean实例，并从earlySingletonObjects、singletonFactories移除Bean实例
        addSingleton(beanName,singletonObject);
                 }
            }
        return singletonObject;
            }
        }
```
可以看到主要流程就是先标记为singletonsCurrentlyInCreation，再调用singletonFactory工厂对象去创建实例，然后移除singletonsCurrentlyInCreation的标记,
最后加入singletonObjects中(最终创建好并且初始化完成的Bean实例的集合)。<br>
现在我们继续阅读 ObjectFactory<?> singletonFactory的createBean这个方法。

## 3.createBean

createBean方法如下：
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
        //获得Bean的类型class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
        // 方法覆盖
		try {
			mbdToUse.prepareMethodOverrides();
		}
        
		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
           //创建代理对象以替换实际的真实目标对象，进入主要看到postProcessorsBeforeInstantiation和postProcessorsAfterInitialization两个方法。
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		

		try {
            //do开头，真正的创建Bean实例的方法。do开头一般都是真正干活的方法。
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		
	}

```
现在我们来到了真正创建Bean实例的方法doCreateBean。

## 4.doCreateBean
我们看下doCreateBean方法：
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		//实例化Bean
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			//先从factoryBeanInstanceCache，工厂Bean的缓存中去获取
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//里面方法是最终是调用构造器的反射方法实例化Bean
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
        
		// Allow post-processors to modify the merged bean definition.
		//beanPostProcessors处理
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				mbd.markAsPostProcessed();
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//注意看这里，提前缓存单例以便解决循环依赖
		//earlySingletonExposure，是否提前暴露单例
		//isSingleton,单例对象，我们默认创建的是单例
		//allowCircularReferences，允许循环依赖，默认为true
		//isSingletonCurrentlyInCreation，是否当前正在创建，在singletonsCurrentlyInCreation查找。我们再上面getSingleton的
        //beforeSingletonCreation方法将BeanName存入了singletonsCurrentlyInCreation，
		//所以isSingletonCurrentlyInCreation=true。最终earlySingletonExposure=true。
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				//注意此处日志
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//将这个Bean对应singletonFactory工厂实例存入singletonFactories，并且earlySingletonObjects删除这个Bean
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		//初始化过程
		Object exposedObject = bean;
		try {
			//属性填充，依赖注入，这里可以注入@Autowired等当前单例bean实例所依赖的对象
			populateBean(beanName, mbd, instanceWrapper);
			//初始化方法，如init-method，InitializingBean，@PostConstruct等
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		
			
		}

		if (earlySingletonExposure) {
			//从singletonObjects，earlySingletonObjects中获取到Bean
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
		}

		// Register bean as disposable.
		//注册销毁方法DisposableBean，@PreDestroy,destroy-method等
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		

		return exposedObject;
	}
```
这里的执行流程还是比较清晰的，先使用createBeanInstance通过反射创建实例，再加入创建工厂对象加入singletonFactory以便后续提前暴露以解决循环依赖，然后populateBean属性填充和依赖注入
，然后initializeBean各种初始化方法和操作，最后注册销毁方法registerDisposableBeanIfNecessary。BeanWrapper是Bean的一个装饰类，这里不做过多解读。
至此，单例bean的创建流程全部解读完成了。我们回到getSingleton内部的这个方法：
```java
protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```
这里将实例化并且完成初始化完成，可以交付给IoC容器的单例Bean使用并加入了singletonObjects集合（一级缓存）。<br>
关于循环依赖，笔者将在下一篇文章介绍。
# 三、Bean生命周期
## 1.源码探究
经过上面的讲解，我们已经比较详细的了解了Bean的创建过程。我们可以分类关注这些方法或过程:<br>
1.bean自身的方法或操作,先实例化(instantiation)、属性填充(setter/populate)、初始化(initialize)、使用中、销毁。这个Bean层级的。<br>
然后我们再关注:<br>
2.BeanPostProcessor,后置处理器对Bean的自身方法的一些增强。这是容器层级的，一个BeanPostProcessor可以作用于所有的。
以及:<br>
3.Bean实现Aware接口获得应用上下文相关的信息。这个也是Bean层级的。<br>
我们这里先忽略了BeanFactoryPostProcessor，对BeanFactory的增强处理器，关注于与Bean相关的拓展处理。
我们再看下doCreateBean这个方法(做了较大删减)：
```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.1.实例化
		BeanWrapper instanceWrapper = null;
		instanceWrapper = createBeanInstance(beanName, mbd, args);
		
		

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//2,属性填充
			populateBean(beanName, mbd, instanceWrapper);
			//3,初始化
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		// Register bean as disposable.
		try {
			//4.注册销毁方法
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}

		return exposedObject;
	}
```
关注Bean自身，这里创建新的单例Bean的核心的四个步骤。我们阅读BeanPostProcessor和InstantiationAwareBeanPostProcessor如下：<br>
因此我们推断BeanPostProcessor的postProcessBeforeInitialization和postProcessAfterInitialization看名字应该在"initializeBean"前
和后,<br>而InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation和postProcessAfterInstantiation应该在"1.实例化"的
前和后，而Aware接口笔者尚未从从上面源码获知是从何处调用的。<br>
下面笔者从一个实际的实例来探讨Bean生命周期各个过程的处理。
## 2.实例
先创建一个Bean的类型：
```java
package com.homura.bean;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class MyTestBean implements ApplicationContextAware, BeanNameAware, BeanFactoryAware, InitializingBean, DisposableBean {
    private static final Logger LOGGER = LoggerFactory.getLogger(MyTestBean.class);

    private ApplicationContext applicationContext;
    private BeanFactory beanFactory;
    private String beanName;

    public MyTestBean() {
        LOGGER.info("MyTestBean.MyTestBean()");
    }

    @PostConstruct
    public void postConstruct() {
        LOGGER.info("MyTestBean.postConstruct()");
    }

    @PreDestroy
    public void preDestroy() {
        LOGGER.info("MyTestBean.preDestroy()");
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        LOGGER.info("MyTestBean.setApplicationContext()");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
        LOGGER.info("MyTestBean.setBeanFactory()");
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        LOGGER.info("MyTestBean.setBeanName()");
    }

    @Override
    public void destroy() throws Exception {
        LOGGER.info("MyTestBean.destroy()");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        LOGGER.info("MyTestBean.afterPropertiesSet()");
    }

    /**
     * init-method
     */
    public void initMethod() {
        LOGGER.info("MyTestBean.initMethod()");
    }

    /**
     * destroy-method
     */
    public void destroyMethod() {
        LOGGER.info("MyTestBean.destroyMethod()");
    }

    public void doSomething() {
        LOGGER.info("MyTestBean.doSomething()");
    }
}

```

然后创建BeanPostProcessor,

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    private static final Logger LOGGER= LoggerFactory.getLogger(MyBeanPostProcessor.class);

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        LOGGER.info("MyBeanPostProcessor.postProcessBeforeInitialization()");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        LOGGER.info("MyBeanPostProcessor.postProcessAfterInitialization()");
        return bean;
    }
}

```
再创建InstantiationAwareBeanPostProcessor，
```java

@Component
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    private static final Logger LOGGER = LoggerFactory.getLogger(MyInstantiationAwareBeanPostProcessor.class);

    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        LOGGER.info("MyInstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()");
        return InstantiationAwareBeanPostProcessor.super.postProcessBeforeInstantiation(beanClass, beanName);
    }

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        LOGGER.info("MyInstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()");
        return true;
    }

    public Object postProcessProperties(Object bean, String beanName) {
        LOGGER.info("MyInstantiationAwareBeanPostProcessor.postProcessProperties()");
        return bean;
    }
}

```
创建测试主类：
```java
package com.homura.bean;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyTestBeanTest {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
        applicationContext.start();
        MyTestBean myTestBean = applicationContext.getBean(MyTestBean.class);
        myTestBean.doSomething();
        applicationContext.close();
    }
}

```

这里要注意的是BeanPostProcessor/InstantiationAwareBeanPostProcessor是会作用于所有的Bean的。
我们看下日志,关注一下myTestBean的创建过程。
具体如下：
```text
2023-09-27 19:44:30,004|DEBUG|DefaultSingletonBeanRegistry.java:225 |main|Creating shared instance of singleton bean 'myTestBean'
2023-09-27 19:44:30,004|INFO |MyInstantiationAwareBeanPostProcessor.java:14  |main|MyInstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
2023-09-27 19:44:30,004|INFO |         MyTestBean.java:20  |main|MyTestBean.MyTestBean()
2023-09-27 19:44:30,008|INFO |MyInstantiationAwareBeanPostProcessor.java:20  |main|MyInstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
2023-09-27 19:44:30,008|INFO |         MyTestBean.java:49  |main|MyTestBean.setBeanName()
2023-09-27 19:44:30,008|INFO |         MyTestBean.java:43  |main|MyTestBean.setBeanFactory()
2023-09-27 19:44:30,008|INFO |         MyTestBean.java:37  |main|MyTestBean.setApplicationContext()
2023-09-27 19:44:30,008|INFO |MyBeanPostProcessor.java:14  |main|MyBeanPostProcessor.postProcessBeforeInitialization()
2023-09-27 19:44:30,008|INFO |         MyTestBean.java:25  |main|MyTestBean.postConstruct()
2023-09-27 19:44:30,008|INFO |         MyTestBean.java:59  |main|MyTestBean.afterPropertiesSet()
2023-09-27 19:44:30,009|INFO |         MyTestBean.java:66  |main|MyTestBean.initMethod()
2023-09-27 19:44:30,010|INFO |MyBeanPostProcessor.java:20  |main|MyBeanPostProcessor.postProcessAfterInitialization()
2023-09-27 19:44:30,026|INFO |         MyTestBean.java:77  |main|MyTestBean.doSomething()
2023-09-27 19:44:30,027|DEBUG|AbstractApplicationContext.java:1054|main|Closing org.springframework.context.support.ClassPathXmlApplicationContext@353352b6, started on Wed Sep 27 19:44:28 CST 2023
2023-09-27 19:44:30,028|INFO |         MyTestBean.java:30  |main|MyTestBean.preDestroy()
2023-09-27 19:44:30,028|INFO |         MyTestBean.java:54  |main|MyTestBean.destroy()
2023-09-27 19:44:30,028|INFO |         MyTestBean.java:73  |main|MyTestBean.destroyMethod()
```
我们能看到这些方法的执行顺序是：
postProcessBeforeInstantiation()--->MyTestBean(),Bean自己的构造器-->属性填充，依赖注入--->postProcessAfterInstantiation()
-->Aware接口
-->postProcessBeforeInitialization()-->postConstruct()--->afterPropertiesSet()-->init-method--->postProcessAfterInitialization()
--->doSomething()类自己的普通方法,初始化完成，交付给客户端使用
--->preDestroy()-->destroy()-->destroy-method
## 3.Bean生命周期总结
简单总结一下上面的Bean的生命周期。
我们重新阅读一下这个doCreateBean方法：
```java
```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.1.实例化
		BeanWrapper instanceWrapper = null;
		instanceWrapper = createBeanInstance(beanName, mbd, args);
		
		

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//2,属性填充
			populateBean(beanName, mbd, instanceWrapper);
			//3,初始化
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		// Register bean as disposable.
		try {
			//4.注册销毁方法
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}

		return exposedObject;
	}
```

按照上面的测试的结论，首先在Bean的实例化和属性填充前后分别插入了postProcessBeforeInstantiation和postProcessAfterInstantiation处理。<br>
然后，在initializeBean初始化前后分别插入了postProcessBeforeInitialization和postProcessAfterInitialization处理。<br>
最后，Aware作用于postProcessAfterInstantiation和postProcessBeforeInitialization之间，也就是实例化完成，尚未初始化的时候。<br>
总体总结如下:

![BeanLifecylce](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-26-spring_source_code_reading_6/BeanLifecylce.png)

# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)  <br>
