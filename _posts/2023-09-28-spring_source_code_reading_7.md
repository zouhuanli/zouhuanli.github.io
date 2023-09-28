---
layout: post
title: "Spring源码阅读七:循环依赖问题"
date: 2023-09-28
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第七篇文章，本篇文章将从源码角度去探讨Spring循环依赖的解决方式。<br>
本文源码地址为:[https://github.com/zouhuanli/SpringCircularReferenceDemo.git](https://github.com/zouhuanli/SpringCircularReferenceDemo.git).<br>
先看一个简单的案例。

# 一、案例
创建beanA：
```java
@Component
public class Alpha {
    @Autowired
    private Beta beta;

    public void a() {
        System.out.println("Alpha.a(): beta=" + this.beta);
    }

    public void setBeta(Beta beta) {
        this.beta = beta;
    }
}
```
再创建beanB:
```java
@Component
public class Beta {
    @Autowired
    private Alpha alpha;

    public void b() {
        System.out.println("Beta.b(): alpha=" + this.alpha);
    }

    public void setAlpha(Alpha alpha) {
        this.alpha = alpha;
    }
}
```
,然后测试的主类；
```java
public class AnnotationContextTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        context.start();
        context.getBean(Beta.class).b();
        //报错，AnnotationConfigApplicationContext无法获取XML配置的Bean
        Alpha alphaBean = context.getBean("alphaBean", Alpha.class);
    }
}

```
,运行一下测试主类，查找一下名字为alpha的Bean。如下：
```text
2023-09-28 19:55:53,329|DEBUG|DefaultSingletonBeanRegistry.java:225 |main|Creating shared instance of singleton bean 'alpha'
2023-09-28 19:55:53,329|TRACE|AbstractAutowireCapableBeanFactory.java:485 |main|Creating instance of bean 'alpha'
2023-09-28 19:55:53,331|TRACE|AbstractAutowireCapableBeanFactory.java:588 |main|Eagerly caching bean 'alpha' to allow for resolving potential circular references
2023-09-28 19:55:53,334|DEBUG|DefaultSingletonBeanRegistry.java:225 |main|Creating shared instance of singleton bean 'beta'
2023-09-28 19:55:53,334|TRACE|AbstractAutowireCapableBeanFactory.java:485 |main|Creating instance of bean 'beta'
2023-09-28 19:55:53,335|TRACE|AbstractAutowireCapableBeanFactory.java:588 |main|Eagerly caching bean 'beta' to allow for resolving potential circular references
2023-09-28 19:55:53,335|TRACE|AbstractBeanFactory.java:252 |main|Returning eagerly cached instance of singleton bean 'alpha' that is not fully initialized yet - a consequence of a circular reference
2023-09-28 19:55:53,335|TRACE|AutowiredAnnotationBeanPostProcessor.java:630 |main|Autowiring by type from bean name 'beta' to bean named 'alpha'
2023-09-28 19:55:53,336|TRACE|AbstractAutowireCapableBeanFactory.java:522 |main|Finished creating instance of bean 'beta'
2023-09-28 19:55:53,336|TRACE|AutowiredAnnotationBeanPostProcessor.java:630 |main|Autowiring by type from bean name 'alpha' to bean named 'beta'
2023-09-28 19:55:53,336|TRACE|AbstractAutowireCapableBeanFactory.java:522 |main|Finished creating instance of bean 'alpha'
2023-09-28 19:55:53,336|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'beta'
2023-09-28 19:55:53,338|TRACE|EventListenerMethodProcessor.java:172 |main|No @EventListener annotations found on bean class: com.homura.app.config.AppConfig$$SpringCGLIB$$0
2023-09-28 19:55:53,339|TRACE|EventListenerMethodProcessor.java:172 |main|No @EventListener annotations found on bean class: com.homura.app.bean.Alpha
2023-09-28 19:55:53,339|TRACE|EventListenerMethodProcessor.java:172 |main|No @EventListener annotations found on bean class: com.homura.app.bean.Beta
2023-09-28 19:55:53,339|TRACE|AbstractApplicationContext.java:866 |main|No 'lifecycleProcessor' bean, using [DefaultLifecycleProcessor]
2023-09-28 19:55:53,340|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'lifecycleProcessor'
2023-09-28 19:55:53,345|TRACE|SpringFactoriesLoader.java:202 |main|Loaded [org.springframework.beans.BeanInfoFactory] names: []
2023-09-28 19:55:53,346|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'lifecycleProcessor'
2023-09-28 19:55:53,346|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'beta'
Beta.b(): alpha=com.homura.app.bean.Alpha@52045dbe
```
我们注意这两行日志，
<strong>"2023-09-28 19:55:53,331|TRACE|AbstractAutowireCapableBeanFactory.java:588 |main|Eagerly caching bean 'alpha' to allow for resolving potential circular references"</strong>,
提前缓存暴露alpha,以便解决循环依赖。定位到源码：
```java
	// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```
然后，注意这行日志<strong>"2023-09-28 19:55:53,335|TRACE|AbstractBeanFactory.java:252 |main|Returning eagerly cached instance of singleton bean 'alpha' that is not fully initialized yet - a consequence of a circular reference"</strong>,
返回缓存在三级缓存(指singletonFactories)的alpha，提供给beta注入，以便完成beta的创建。我们同样定位到源码:
```java
// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
```
以及:
```java
@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
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
从测试验证的结果看，上面就是关于Spring解决循环依赖的关键的两个方法了。下面笔者再从源码角度去解读一下整个的流程。
# 二、源码解读
笔者在前一篇的文章上这里再简化总结一下Spring创建Bean的流程，主要关注一些主方法以及与本文主题相关的方法。
1. doGetBean
```java
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		String beanName = transformedBeanName(name);
		Object beanInstance;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

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
	}
```
2. getSingleton(String beanName)
```java

	/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
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
3. getSingleton(String beanName, ObjectFactory<?> singletonFactory)
```java
	/**
	 * Return the (raw) singleton object registered under the given name,
	 * creating and registering a new one if none registered yet.
	 * @param beanName the name of the bean
	 * @param singletonFactory the ObjectFactory to lazily create the singleton
	 * with, if necessary
	 * @return the registered singleton object
	 */
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				try {
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}

				finally {
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}

```
4. createBean和doCreateBean
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		
		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		
		

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		
		return exposedObject;
	}
```
笔者现源码角度分析BeanA和BeanB的执行流程，a指alpha，a指beta。假定a先加载先创建,并且容器的bean集合是空的。<br>
1>a的过程：先创建a。a去getSingleton查询三级缓存(singletonObjects-->earlySingletonObjects--->singletonFactories顺序)获取，因为缓存集合都是空的。没有，则开始创建流程。<br>
2>a的过程：进入getSingleton方法，先标记a为创建中，加入singletonsCurrentlyInCreation集合。<br>
3>a的过程：调用singletonFactory创建a，进入createBean方法，再进入doCreateBean方法。<br>
4>a的过程：<strong>doCreateBean是真正创建bean实例的方法，分为四步：实例化（调用构造器反射创建对象）、属性填充和依赖注入、初始化、注册销毁方法。对于单例在实例化之后，马上加入singletonFactories三级缓存中。<br>
存入的key是bean的name，值是"() -> getEarlyBeanReference(beanName, mbd, bean)"这样一个的singletonFactory的的工厂方法。</strong><br>
5>a的过程：然后a开始属性填充和依赖注入，按type或者name注入时候发现依赖b。则开始getBean(b),创建b。<br>
6>b的过程：b的创建过程同1步~5步。b开始依赖注入的时候又发现依赖a，则开始getBean(a)。<br>
7>b的过程：<strong>b获取a的时候，会从getSingleton这里从singletonFactories这个三级缓存中获取到a，此时a只是实例化、还未属性填充和依赖注入，尚未初始化完成，是一个半成品。</strong><br>
8>b的过程:b此时完成了属性填充和依赖注入，然后初始化完成，注册销毁方法完成。然后b此时是一个完全对象了，加入到了一级缓存singletonObjects中，并且持有了a的引用。<br>
9>a的过程：8步之后b已经完成创建完成了，并加入一级缓存singletonObjects。此时a在populateBean方法也完成了然后退出方法，并且持有了b的引用，b此时是完整对象。<br>
10>a的过程：a初始化(initializeBean方法)完成,因为上面在7步这里将a的缓存从singletonFactories删除，升级到二级缓存earlySingletonObjects，所以earlySingletonObjects存有a的引用。<br>
11>a的过程：退出doCreateBean和createBean方法，进入getSingleton(String beanName, ObjectFactory<?> singletonFactory)这个方法,在singletonsCurrentlyInCreation移除a，<br>
然后从earlySingletonObjects、singletonFactories删除a，并加入到一级缓存singletonObjects中。<br>

至此，a和b都创建完成，并且初始化完成了，然后都加入了一级缓存singletonObjects，后续get就之间从里面获取，不会走创建流程。<br>

可以看出最核心的部分就是使用了三级缓存和提前加入到singletonFactories提前曝光对象的引用。也就是4步和7步。<br>
下面看下debug流程的一些核心步骤数据：<br>
a加入singletonFactories中:
![addATo](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-28-spring_source_code_reading_7/addATo.png)

b加入singletonFactories中:
![addBTo](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-28-spring_source_code_reading_7/addBTo.png)

b的注入阶段去获取a：
![whenBGetA](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-28-spring_source_code_reading_7/whenBGetA.png)


# 三、总结和注意点
先总结一下循环依赖的解决过程如下,图片不是我画的,和我的源码解读的流程一样。<br>图片来自于"https://mp.weixin.qq.com/s/Ye0qCCnffYdhVKSiMWOllg"


![circularReference](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-28-spring_source_code_reading_7/circularReference.png)

一些注意点：<br>
1.没有解决构造器的循环依赖。官方解释如下：

```text
If you use predominantly constructor injection, it is possible to create an unresolvable
circular dependency scenario.
For example: Class A requires an instance of class B through constructor injection, and class B
requires an instance of class A through constructor injection. If you configure beans for
classes A and B to be injected into each other, the Spring IoC container detects this circular
reference at runtime, and throws a BeanCurrentlyInCreationException.
One possible solution is to edit the source code of some classes to be configured by setters
rather than constructors. Alternatively, avoid constructor injection and use setter injection
only. In other words, although it is not recommended, you can configure circular
dependencies with setter injection.
Unlike the typical case (with no circular dependencies), a circular dependency between bean
A and bean B forces one of the beans to be injected into the other prior to being fully
initialized itself (a classic chicken-and-egg scenario).
```

2.没有解决原型和scope的循环依赖，因为根本没有加入三级缓存之中。

# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)  <br>
3.https://mp.weixin.qq.com/s/Ye0qCCnffYdhVKSiMWOllg <br>
4.《Spring Framew Documentation》(Version 6.0.8)
