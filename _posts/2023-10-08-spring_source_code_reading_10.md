---
layout: post
title: "Spring源码阅读十:AOP代理对象的创建流程"
date: 2023-10-08
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十篇文章，本篇文章解读真正创建AOP代理对象的源码，探究Spring如何对需要增强的原始对象进行增强操作并生成代理对象。<br>
上一篇文章我们研究了Aop切面配置的标签和注解如何被解析为Aop的beanDefinition，并生成封装了切面方法+切入点的Advisor对象，并生成Advisor的实例
bean，我们特别关注了AnnotationAwareAspectJAutoProxyCreator这个类。<br>
前一篇文件解读了AnnotationAwareAspectJAutoProxyCreator的postProcessBeforeInstantiation方法，这个方法生成了XML配置的aop的Advisor
的实例Bean，以及解析@Aspect类获取切面方法注册切面信息，并生成对应的Advisor对象。
这里对前一篇文章做一下补充，注册"org.springframework.aop.config.internalAutoProxyCreator"这个用于AOP切面解析和Advisor实例创建的Bean
的时候，会按照下面的优先级选择，越后越高优先级，所以AnnotationAwareAspectJAutoProxyCreator的注册信息会覆盖AspectJAwareAdvisorAutoProxyCreator
的注册信息，同时注册时候会选择AnnotationAwareAspectJAutoProxyCreator。
```java

	/**
	 * Stores the auto proxy creator classes in escalation order.
	 */
	private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

	static {
		// Set up the escalation list...
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
	}
```
下面开始阅读真正创建代理对象的源码<strong>AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法</strong>。<br>
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法

我们先进入方法postProcessAfterInitialization：

```java
@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

然后阅读wrapIfNecessary方法：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            //advisedBeans 的key是Bean的name，value是是否需要切面增强
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
        //获取到切面类+切入点的封装类advisor的列表
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
            //生成代理对象
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
继续阅读getAdvicesAndAdvisorsForBean方法：

```java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

        List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
        //没有适用的Advisor对象，不需要创建代理对象
        if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
        }
        //Advisor对象,拦截器列表
        return advisors.toArray();
        }
```
然后是findEligibleAdvisors,找到适用的advisors列表。
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //上一篇文件讲到的所有的Advisor对象
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
        //寻找适用的
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
            //排序,按照Order
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```
我们在看下寻找适用的Advisor列表的方法：
```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new ArrayList<>();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
```
candidateAdvisors列表如下：


我们继续深入，阅读canApply方法：
```java
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher iamm) {
			introductionAwareMethodMatcher = iamm;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}

```
![introductionAwareMethodMatcher](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_10/introductionAwareMethodMatcher.png)
如图,这里我们看introductionAwareMethodMatcher是AspectJExpressionPointcut切入点的匹配器，去遍历目标类的所有方法,匹配符合切入点表达式的方法。
我们的目标类UserController符合我们示例工程的切入点表达式,得到canApply的返回值为true。

最后得到所有匹配的切入点集合eligibleAdvisors，然后按照order排序。
```java

@Override
	protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
		List<PartiallyComparableAdvisorHolder> partiallyComparableAdvisors = new ArrayList<>(advisors.size());
		for (Advisor advisor : advisors) {
			partiallyComparableAdvisors.add(
					new PartiallyComparableAdvisorHolder(advisor, DEFAULT_PRECEDENCE_COMPARATOR));
		}
		List<PartiallyComparableAdvisorHolder> sorted = PartialOrder.sort(partiallyComparableAdvisors);
		if (sorted != null) {
			List<Advisor> result = new ArrayList<>(advisors.size());
			for (PartiallyComparableAdvisorHolder pcAdvisor : sorted) {
				result.add(pcAdvisor.getAdvisor());
			}
			return result;
		}
		else {
			return super.sortAdvisors(advisors);
		}
	}
```
最终拿到所有的拦截器:
```java

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
```
然后createProxy开始创建代理对象。

# 二、创建代理对象createProxy方法

我们进入createProxy方法，再进入buildProxy方法：

```java
	private Object buildProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource, boolean classOnly) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory clbf) {
			AutoProxyUtils.exposeTargetClass(clbf, beanName, beanClass);
		}
        //代理对象的工厂类
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);
        //代理类，proxy-target-class
		if (proxyFactory.isProxyTargetClass()) {
			// Explicit handling of JDK proxy targets and lambdas (for introduction advice scenarios)
			if (Proxy.isProxyClass(beanClass) || ClassUtils.isLambdaClass(beanClass)) {
				// Must allow for introductions; can't just set interfaces to the proxy's interfaces only.
				for (Class<?> ifc : beanClass.getInterfaces()) {
					proxyFactory.addInterface(ifc);
				}
			}
		}
		else {
			// No proxyTargetClass flag enforced, let's apply our default checks...
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
        //Advisor适配
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		// Use original ClassLoader if bean class not locally loaded in overriding class loader
        //类加载器
		ClassLoader classLoader = getProxyClassLoader();
		if (classLoader instanceof SmartClassLoader smartClassLoader && classLoader != beanClass.getClassLoader()) {
			classLoader = smartClassLoader.getOriginalClassLoader();
		}
        //classOnly-只获取目标类的类型
        //proxyFactory.getProxy,真正创建代理对象的方法
		return (classOnly ? proxyFactory.getProxyClass(classLoader) : proxyFactory.getProxy(classLoader));
	}
```
这里我们可以看到创建代理对象是交给了proxyFactory这个工厂类去创建的。<br>
继续阅读:
```java
/**
	 * Create a new proxy according to the settings in this factory.
	 * <p>Can be called repeatedly. Effect will vary if we've added
	 * or removed interfaces. Can add and remove interceptors.
	 * <p>Uses the given class loader (if necessary for proxy creation).
	 * @param classLoader the class loader to create the proxy with
	 * (or {@code null} for the low-level proxy facility's default)
	 * @return the proxy object
	 */
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
```
```java
@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
这里分为两步，先创建AopProxy，再创建具体的代理对象。AopProxy的官方解释是"Delegate interface for a configured AOP proxy, allowing 
for the creation of actual proxy objects"，创建实际代理对象的一个委托对象。有JdkDynamicAopProxy、ObjenesisCglibAopProxy（CglibAopProxy的子类）。
接下来就是使用Jdk动态代理或者Cglib动态代理生成实际的代理对象了。

# 三、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)
