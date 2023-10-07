---
layout: post
title: "Spring源码阅读九:AOP切面的解析和创建"
date: 2023-10-07
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第九篇文章，本篇文章着重介绍AOP切面(Aspect)等元素的解析和切面类创建。<br>
我们借鉴IoC的初始化过程，我们先推断下面元素也会按照XmlXXXBeanDefinitionReader读取为XXXbean的定义信息，每个切面方法会和切面类aspect封装在一起或分开。
```xml
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
下面笔者从源码开始探究AOP切面的解析和创建流程。本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、Aop配置标签的解析
## 1.入口
我们在IoC容器初始化过程那篇文章，有这段代码:
```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element ele) {
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```
我们通过parseCustomElement方法进入AopNamespaceHandler类:
```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.5+ XSDs
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace in 2.5+
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```
这里就是解析&lt;aop:config>、&lt;aop:aspectj-autoproxy/>、&lt;aop:scoped-proxy/>、&lt;aop:spring-configured/>等标签的解析类了。

## 2.ConfigBeanDefinitionParser
我们继续研读ConfigBeanDefinitionParser这个解析AOP配置的类。我们发现，到AopConfigUtils里面:
```java
	@Nullable
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source);
	}
```
这里注册了<strong>AspectJAwareAdvisorAutoProxyCreator</strong>。继续解读：
```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
		CompositeComponentDefinition compositeDef =
				new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
		parserContext.pushContainingComponent(compositeDef);

		configureAutoProxyCreator(parserContext, element);

		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			switch (localName) {
				case POINTCUT -> parsePointcut(elt, parserContext);
				case ADVISOR -> parseAdvisor(elt, parserContext);
				case ASPECT -> parseAspect(elt, parserContext);
			}
		}

		parserContext.popAndRegisterContainingComponent();
		return null;
	}
```
这里分别解析了pointcut、advisor、aspect等元素，继续跟踪parseAspect方法,解析&lt;aop:aspect>标签如下：

![AopAspect](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/AopAspect.png)


继续跟踪，我们看parseAdvice方法，这里解析了切面方法，如下：

![parseAdvice](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/parseAdvice.png)

我们发现切面方法被解析成的类型是AspectJPointcutAdvisor。AspectJPointcutAdvisor内封装了AbstractAspectJAdvice(五种通知)，Pointcut。
现在我们看下完整的解析完成的aspect标签的信息：

![aspectComponentDefinition](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/aspectComponentDefinition.png)

现在aspect标签就已经解析完成了，拿到切面类和切面方法的定义信息。<br>
我们再看下AspectJAutoProxyBeanDefinitionParser。
## 3.AspectJAutoProxyBeanDefinitionParser
<aop:aspectj-autoproxy/>由AspectJAutoProxyBeanDefinitionParser解析。
由我们阅读源码发现，其就注册了AnnotationAwareAspectJAutoProxyCreator和设置proxy-target-class、expose-proxy。
```java

	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}
```
我们上面解析config标签时候注册了AspectJAwareAdvisorAutoProxyCreator。我们看到AnnotationAwareAspectJAutoProxyCreator是AspectJAwareAdvisorAutoProxyCreator子类，
先我们重点解读一下AnnotationAwareAspectJAutoProxyCreator，该类的关系图如下:

![AnnotationAwareAspectJAutoProxyCreator](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/AnnotationAwareAspectJAutoProxyCreator.png)

我们看到的顶级接口有Aware、BeanPostProcessor，我们在Bean的生命周期那篇文章有提到：

```text
Aware作用于postProcessAfterInstantiation和postProcessBeforeInitialization之间，也就是实例化完成，尚未初始化的时候。
Bean的实例化和属性填充前后分别插入了postProcessBeforeInstantiation和postProcessAfterInstantiation处理。
在initializeBean初始化前后分别插入了postProcessBeforeInitialization和postProcessAfterInitialization处理。
```

因此我们这里重点关注一些postProcessBeforeXXX和postProcessAfterXXX。我们跟踪源码，进入AbstractAutoProxyCreator。<br>
先研究postProcessBeforeInstantiation方法：
```java
@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}
```
我们回顾Bean创建流程中的这段代码:
```java
// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
//给BeanPostProcessors一个机会/时机去创建和返回一个代理对象，而不是真实的目标对象。
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
            
            
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        Class<?> targetType = determineTargetType(beanName, mbd);
        if (targetType != null) {
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
        }
        }
        mbd.beforeInstantiationResolved = (bean != null);
        }
        return bean;
        }
```
这里使用到了beanPostProcessorsBeforeInstantiation和beanPostProcessorsAfterInitialization来创建代理对象。
我们开始阅读<stong>AbstractAutoProxyCreator</strong>的<stong>postProcessBeforeInstantiation</strong>。
# 二、AnnotationAwareAspectJAutoProxyCreator
先看下源码:
```java
    //在Bean实例化(构造器)之前就调用的后置处理代码
    @Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
            //是否是基础类
            //是否应该跳过
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}
        //设置了custom TargetSource则在这里就创建代理对象。我们示例工程不涉及这个。
		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}

```
是否基础类,如org.springframework.aop.aspectj.AspectJAroundAdvice是Advice类，以及AnnotationAwareAspectJAutoProxyCreator实现了AopInfrastructureBean：
```java
protected boolean isInfrastructureClass(Class<?> beanClass) {
		boolean retVal = Advice.class.isAssignableFrom(beanClass) ||
				Pointcut.class.isAssignableFrom(beanClass) ||
				Advisor.class.isAssignableFrom(beanClass) ||
				AopInfrastructureBean.class.isAssignableFrom(beanClass);
		if (retVal && logger.isTraceEnabled()) {
			logger.trace("Did not attempt to auto-proxy infrastructure class [" + beanClass.getName() + "]");
		}
		return retVal;
	}
```
是否应该跳过，
```java
@Override
	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		// TODO: Consider optimization by caching the list of the aspect names
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		for (Advisor advisor : candidateAdvisors) {
			if (advisor instanceof AspectJPointcutAdvisor pointcutAdvisor &&
					pointcutAdvisor.getAspectName().equals(beanName)) {
				return true;
			}
		}
		return super.shouldSkip(beanClass, beanName);
	}
```
一路debug跟踪执行流程，进入findAdvisorBeans方法：
```java
public List<Advisor> findAdvisorBeans() {
		// Determine list of advisor bean names, if not cached already.
		String[] advisorNames = this.cachedAdvisorBeanNames;
		if (advisorNames == null) {
			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the auto-proxy creator apply to them!
			advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.beanFactory, Advisor.class, true, false);
			this.cachedAdvisorBeanNames = advisorNames;
		}
		if (advisorNames.length == 0) {
			return new ArrayList<>();
		}

		List<Advisor> advisors = new ArrayList<>();
		for (String name : advisorNames) {
			if (isEligibleBean(name)) {
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isTraceEnabled()) {
						logger.trace("Skipping currently created advisor '" + name + "'");
					}
				}
				else {
					try {
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						Throwable rootCause = ex.getMostSpecificCause();
						if (rootCause instanceof BeanCurrentlyInCreationException bce) {
							String bceBeanName = bce.getBeanName();
							if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
								if (logger.isTraceEnabled()) {
									logger.trace("Skipping advisor '" + name +
											"' with dependency on currently created bean: " + ex.getMessage());
								}
								// Ignore: indicates a reference back to the bean we're trying to advise.
								// We want to find advisors other than the currently created bean itself.
								continue;
							}
						}
						throw ex;
					}
				}
			}
		}
		return advisors;
	}
```
这里获取到之前解析的(存入BeanFactory里面)所有的通知类的Bean，将切面方法的定义信息(definition)转换为实际的实例(Bean)如下：

![cachedAdvisorBeanNames](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/cachedAdvisorBeanNames.png)

![advisors](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/advisors.png)



我们再调试一下这行代码:
```java
// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
```
aspectJAdvisorsBuilder是的org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator$BeanFactoryAspectJAdvisorsBuilderAdapter，这样一个<br>
AnnotationAwareAspectJAutoProxyCreator的适配器。我们继续解读buildAspectJAdvisors方法：
```java

	/**
	 * Look for AspectJ-annotated aspect beans in the current bean factory,
	 * and return to a list of Spring AOP Advisors representing them.
	 * <p>Creates a Spring Advisor for each AspectJ advice method.
	 * @return the list of {@link org.springframework.aop.Advisor} beans
	 * @see #isEligibleBean
	 */
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new ArrayList<>();
					aspectNames = new ArrayList<>();
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
					for (String beanName : beanNames) {
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
						Class<?> beanType = this.beanFactory.getType(beanName, false);
						if (beanType == null) {
							continue;
						}
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new ArrayList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```
这段代码结合注释和源码是解析@AspectJ注解的bean，然后获取切面方法，存入缓存aspectBeanNames(单例)下次直接获取，将切面方法+切入点封装在一起的Advisor组成列表返回。<br>

```java
if (this.advisorFactory.isAspect(beanType))
```
这里的判断很简单就看是否有@AspectJ注解，以及不是以"ajc$”(应该是某些框架自己的类型)开头。如果全部aop配置都在XML，没有使用@Aspect注解，毫无意外的这里是一个空集合。
```text
aspectNames:size=0
```

我们这里按注解的方式调试一下此处的代码。跟踪进入getAdvisor方法:
```java

	@Override
	@Nullable
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
        //获取切入点
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}

		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
```
这里测试发现，这里的Advisor类型是InstantiationModelAwarePointcutAdvisorImpl：

![InstantiationModelAwarePointcutAdvisorImpl](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-07-spring_source_code_reading_9/InstantiationModelAwarePointcutAdvisorImpl.png)

返回的advisors依旧是包含切面方法+切入点封装在一起的Advisor的列表,实例工程的是五个Advisor对象。<br>
至此所有的AOP配置(XML和注解)的都已经解析完成，并注册为Advisor对象的列表。可以供后续拦截需要生成代理对象的Bean的时候使用。

# 三、补充
在纯注解的配置中@EnableAspectJAutoProxy有这样一个配置:

```java
@Import(AspectJAutoProxyRegistrar.class)
```
我们阅读一下AspectJAutoProxyRegistrar的代码：
```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * Register, escalate, and configure the AspectJ auto proxy creator based on the value
	 * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
	 * {@code @Configuration} class.
	 */
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}

}
```
继续阅读发现其仍然是注册了AnnotationAwareAspectJAutoProxyCreator。其执行过程是ConfigurationClassBeanDefinitionReader的loadBeanDefinitionsFromRegistrars方法。<br>
本文更加关注AnnotationAwareAspectJAutoProxyCreator，前序流程这里不做过多探究。

# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)
