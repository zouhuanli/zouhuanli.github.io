---
layout: post
title: "Spring源码阅读十七:事务元素的解读过程"
date: 2023-10-18
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十七篇文章，本文着重解读事务元素的解析过程和事务代理对象的创建过程。<br>
Spring事务也是是使用AOP创建代理来实现的，之前解读AOP标签的处理过程：解析AOP标签，注册Advisor(包含Pointcut切面方法和Advice通知/增强处理),然后
在Bean的postProcessAfterInitialization处理过程中,由AbstractAutoProxyCreator查询所有的Advisor对象并匹配需要创建代理的bean，进行创建代理对象。<br>

Spring使用&lt;tx:advice/>元素开启@Transaction注解事务，mode=proxy是默认的代理模式。<br>

```xml

<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" mode="proxy"/>
```

我们从TxNamespaceHandler这个类开始读起，这个类解析了&lt;tx:advice/>、&lt;tx:annotation-driven/>、&lt;tx:
jta-transaction-manager/&gt;这三个配置事务的元素标签。

本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、事务元素配置解析

先看下TxNamespaceHandler这个类。

```java
public class TxNamespaceHandler extends NamespaceHandlerSupport {

	static final String TRANSACTION_MANAGER_ATTRIBUTE = "transaction-manager";

	static final String DEFAULT_TRANSACTION_MANAGER_BEAN_NAME = "transactionManager";


	static String getTransactionManagerName(Element element) {
		return (element.hasAttribute(TRANSACTION_MANAGER_ATTRIBUTE) ?
				element.getAttribute(TRANSACTION_MANAGER_ATTRIBUTE) : DEFAULT_TRANSACTION_MANAGER_BEAN_NAME);
	}


	@Override
	public void init() {
		registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
	}

}

```
这里分别注册了TxAdviceBeanDefinitionParser、AnnotationDrivenBeanDefinitionParser、JtaTransactionManagerBeanDefinitionParser。<br>
继续进入AnnotationDrivenBeanDefinitionParser。
```java
@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
    //注册事件监听器的工厂
		registerTransactionalEventListenerFactory(parserContext);
		String mode = element.getAttribute("mode");
		if ("aspectj".equals(mode)) {
			// mode="aspectj"
			registerTransactionAspect(element, parserContext);
			if (ClassUtils.isPresent("jakarta.transaction.Transactional", getClass().getClassLoader())) {
				registerJtaTransactionAspect(element, parserContext);
			}
		}
		else {
			// mode="proxy",默认的代理模式，本例的mode=proxy
			AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
		}
		return null;
	}
```

然后再阅读AnnotationDrivenBeanDefinitionParser$AopAutoProxyConfigurer#configureAutoProxyCreator方法：

```java
private static class AopAutoProxyConfigurer {

		public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
                        //这里注册名字为"org.springframework.aop.config.internalAutoProxyCreator"的代理创建器，
                        //有InfrastructureAdvisorAutoProxyCreator、AspectJAwareAdvisorAutoProxyCreator、AnnotationAwareAspectJAutoProxyCreator三个，后面的优先级更高会覆盖前面的
			AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);

			String txAdvisorBeanName = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME;
			if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
				Object eleSource = parserContext.extractSource(element);

				// Create the TransactionAttributeSource definition.
                //注册AnnotationTransactionAttributeSource
				RootBeanDefinition sourceDef = new RootBeanDefinition(
						"org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
				sourceDef.setSource(eleSource);
				sourceDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);

				// Create the TransactionInterceptor definition.
                //注册事务拦截器TransactionInterceptor，核心
				RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
				interceptorDef.setSource(eleSource);
				interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				registerTransactionManager(element, interceptorDef);
				interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
				String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);

				// Create the TransactionAttributeSourceAdvisor definition.
                //创建Advisor对象
				RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
				advisorDef.setSource(eleSource);
				advisorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
				advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
				if (element.hasAttribute("order")) {
					advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
				}
				parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);

				CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
				compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
				compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
				compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
				parserContext.registerComponent(compositeDef);
			}
		}
	}
```

上面注册了三个Bean：AnnotationTransactionAttributeSource、TransactionInterceptor、BeanFactoryTransactionAttributeSourceAdvisor这三个都是基础框架Bean,
在AbstractAutoProxyCreator创建代理对象时候会直接shouldSkip返回true跳过。<br>
我们先看下这个三个类型的类关系图。
AnnotationTransactionAttributeSource：
![AnnotationTransactionAttributeSource](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/AnnotationTransactionAttributeSource.png)

TransactionInterceptor:
![TransactionInterceptor](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/TransactionInterceptor.png)

BeanFactoryTransactionAttributeSourceAdvisor:
![BeanFactoryTransactionAttributeSourceAdvisor](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/BeanFactoryTransactionAttributeSourceAdvisor.png)

我们进入AnnotationTransactionAttributeSource的这些方法:
```java
@Override
	public boolean isCandidateClass(Class<?> targetClass) {
		for (TransactionAnnotationParser parser : this.annotationParsers) {
			if (parser.isCandidateClass(targetClass)) {
				return true;
			}
		}
		return false;
	}

	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Class<?> clazz) {
		return determineTransactionAttribute(clazz);
	}

	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Method method) {
		return determineTransactionAttribute(method);
	}

	/**
	 * Determine the transaction attribute for the given method or class.
	 * <p>This implementation delegates to configured
	 * {@link TransactionAnnotationParser TransactionAnnotationParsers}
	 * for parsing known annotations into Spring's metadata attribute class.
	 * Returns {@code null} if it's not transactional.
	 * <p>Can be overridden to support custom annotations that carry transaction metadata.
	 * @param element the annotated method or class
	 * @return the configured transaction attribute, or {@code null} if none was found
	 */
	@Nullable
	protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
		for (TransactionAnnotationParser parser : this.annotationParsers) {
			TransactionAttribute attr = parser.parseTransactionAnnotation(element);
			if (attr != null) {
				return attr;
			}
		}
		return null;
	}

```
这里调用TransactionAnnotationParser判断目标类是否需要做事务增强处理，是否需要创建事务属性,并解析事务属性(@Transactional注解)。我们继续进入SpringTransactionAnnotationParser。
```java


/**
 * Strategy implementation for parsing Spring's {@link Transactional} annotation.
 *
 * @author Juergen Hoeller
 * @author Mark Paluch
 * @since 2.5
 */
@SuppressWarnings("serial")
public class SpringTransactionAnnotationParser implements TransactionAnnotationParser, Serializable {

    //判断是否需要事务增强处理，是否是创建事务代理的目标类.本例isCandidateClass(UserServiceImpl.class)=true
	@Override
	public boolean isCandidateClass(Class<?> targetClass) {
		return AnnotationUtils.isCandidateClass(targetClass, Transactional.class);
	}
    
    //解析事务属性,如Propagation、Isolation、rollbackFor等
	@Override
	@Nullable
	public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
				element, Transactional.class, false, false);
		if (attributes != null) {
			return parseTransactionAnnotation(attributes);
		}
		else {
			return null;
		}
	}

	public TransactionAttribute parseTransactionAnnotation(Transactional ann) {
		return parseTransactionAnnotation(AnnotationUtils.getAnnotationAttributes(ann, false, false));
	}
    //解析各种事务属性
	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());

		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		String timeoutString = attributes.getString("timeoutString");
		Assert.isTrue(!StringUtils.hasText(timeoutString) || rbta.getTimeout() < 0,
				"Specify 'timeout' or 'timeoutString', not both");
		rbta.setTimeoutString(timeoutString);

		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));
		rbta.setLabels(Set.of(attributes.getStringArray("label")));

		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
    

}

```
事务属性如下：

![rbta](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/rbta.png)

有了事务属性之后，TransactionAttributeSourcePointcut的matches方法就可以匹配到需要创建事务代理的目标对象。

```java
@Override
	public boolean matches(Method method, Class<?> targetClass) {
		return (this.transactionAttributeSource == null ||
				this.transactionAttributeSource.getTransactionAttribute(method, targetClass) != null);
	}
```

我们再看BeanFactoryTransactionAttributeSourceAdvisor这个类型。
```java
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut();


	/**
	 * Set the transaction attribute source which is used to find transaction
	 * attributes. This should usually be identical to the source reference
	 * set on the transaction interceptor itself.
	 * @see TransactionInterceptor#setTransactionAttributeSource
	 */
	public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
		this.pointcut.setTransactionAttributeSource(transactionAttributeSource);
	}

	/**
	 * Set the {@link ClassFilter} to use for this pointcut.
	 * Default is {@link ClassFilter#TRUE}.
	 */
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}
```
TransactionAttributeSourcePointcut持有TransactionAttributeSource对象引用,TransactionAttributeSourcePointcut持有TransactionAttributeSource对象引用。
同时BeanFactoryTransactionAttributeSourceAdvisor的“Advice advice”通知对象就是TransactionInterceptor对象。
```java

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
			TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {

		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource);
		advisor.setAdvice(transactionInterceptor);
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}
```
这样的BeanFactoryTransactionAttributeSourceAdvisor和其他对象的关系就清楚了：
![BeanFactoryTransactionAttributeSourceAdvisor_TopView](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/BeanFactoryTransactionAttributeSourceAdvisor_TopView.png)

这里把握两条主要的依赖关系线路：
1.BeanFactoryTransactionAttributeSourceAdvisor——TransactionAttributeSourcePointcut——TransactionAttributeSource——AbstractFallbackTransactionAttributeSource
——AnnotationTransactionAttributeSource——TransactionAnnotationParser——SpringTransactionAnnotationParser。<br>
2.BeanFactoryTransactionAttributeSourceAdvisor——AbstractBeanFactoryPointcutAdvisor——Advice——TransactionInterceptor。<br>
至此，事务属性都已经解析完成，并且事务的Advisor(包含Pointcut和Advice对象)都已经解析完成。<br>
接下来就是实际去创建代理对象了，还是在AOP解读文章那提到的AbstractAutoProxyCreator#wrapIfNecessary方法。

# 二、事务代理对象创建

我们继续阅读AbstractAutoProxyCreator创建代理时候的代码AbstractAutoProxyCreator#wrapIfNecessary方法。
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
我们跟踪执行流程，得到事务的切入点Pointcut对象是TransactionAttributeSourcePointcut和Advice增强处理对象是TransactionInterceptor，
以及对两者封装的BeanFactoryTransactionAttributeSourceAdvisor对象。如下：

![specificInterceptors](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/specificInterceptors.png)

对UserService创建的代理的对象为:

![proxy](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/proxy.png)

如何将UserService匹配上对应的BeanFactoryTransactionAttributeSourceAdvisor切面配置类？我们跟踪canApply方法，我进入TransactionAttributeSourcePointcut#matches方法，这里匹配到配置的
事务属性即getTransactionAttribute方法返回对应的事务属性就需要创建事务代理对象。
```java

	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		return (this.transactionAttributeSource == null ||
				this.transactionAttributeSource.getTransactionAttribute(method, targetClass) != null);
	}
```
继续进入AbstractFallbackTransactionAttributeSource#getTransactionAttribute方法,获得方法对应的事务信息：

```java
@Override
	@Nullable
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		// First, see if we have a cached value.
        //cachekey是方法，Method类型
		Object cacheKey = getCacheKey(method, targetClass);
		TransactionAttribute cached = this.attributeCache.get(cacheKey);
		if (cached != null) {
			// Value will either be canonical value indicating there is no transaction attribute,
			// or an actual transaction attribute.
			if (cached == NULL_TRANSACTION_ATTRIBUTE) {
				return null;
			}
			else {
				return cached;
			}
		}
		else {
			// We need to work it out.
        //获得方法的事务信息
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// Put it in the cache.
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			else {
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute dta) {
					dta.setDescriptor(methodIdentification);
					dta.resolveAttributeStrings(this.embeddedValueResolver);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}

```
事务信息如图：
![txAttr](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-18-spring_source_code_reading_17/txAttr.png)

到这里UserService的事务的代理对象就创建完成了,调用代理方法则会执行TransactionInterceptor的增强处理逻辑。<br>
TransactionInterceptor是事务处理的核心方法，真正的实现事务的增强处理，将留到下一篇文章解读。

# 三、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)<br>