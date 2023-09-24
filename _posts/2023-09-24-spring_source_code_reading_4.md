---
layout: post
title: "Spring源码阅读四:Spring IoC容器初始化流程"
date: 2023-09-23
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第四篇文章，本文主要介绍Spring IoC容器的初始化流程，包括BeanFactory和ApplicationContext的初始化流程。

# 一、入口
从这个最简单的例子入手：

```java
  public static void main(String[] args) {
            ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("springmvc-context.xml");
            Beta beta = applicationContext.getBean(Beta.class);
            beta.invokeSigmaMethod();
    }

```
我们看到先是创建了ClassPathXmlApplicationContext，然后按照类型获取Bean。我们继续看到ClassPathXmlApplicationContext的构造方法看下：

```java
  public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[] {configLocation}, true, null);
        }
```
继续往前看：
```java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```
这里设置了父容器和configLocations配置路径。前一篇文章ClassPathXmlApplicationContext继承自AbstractRefreshableConfigApplicationContext，refresh为true,我们看到现在我们看refresh方法。

# 二、refresh方法
refresh是销毁旧容器，刷新容器配置，启动新容器，并发布容器事件的核心方法。读懂refresh方法就读懂了IoC容器的初始化流程了。
我们看到具体代码,这里有删减，忽略了异常、部分分支代码，只看主要代码：
```java
public void refresh() throws BeansException, IllegalStateException {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
        //准备刷新，设置closed、active，初始化了一些应用的监听器的集合
        // Prepare this context for refreshing.
        prepareRefresh();

        //销毁旧的beanFactory，创建新的beanFactory，如果是refresh的ApplicationContext，重新解析、加载、注册BeanDefinition。
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        //准备beanFactory，配置了类加载器、后置处理器、忽略注入的接口等等
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

            //beanFactory的后置处理，空方法，框架预留的拓展点
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);
    
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            //BeanFactory的后置处理器处理
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            //注册Bean的后置处理器
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            //初始化消息资源，i18n等消息
            // Initialize message source for this context.
            initMessageSource();

            //初始化事件广播器
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            //空方法，onRefresh拓展点
            // Initialize other special beans in specific context subclasses.
            onRefresh();

            //注册监听器
            // Check for listener beans and register them.
            registerListeners();

            //实例化Bean(非懒加载的Bean)，看名字这里从BeanDefinitaion创建了所有的注册的bean
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            //完成刷新，清理一些资源，发布事件
            // Last step: publish corresponding event.
            finishRefresh();
      
}
```
可以看到，整体流程还是很清晰的，没有很高的复杂度。然后整个初始化流程中预留了很多拓展点供客户端或者第三方开发者去自己拓展和实现。

# 三、BeanFactory初始化过程
接下来我们看下本篇文章的主要内容，BeanFactory初始化过程，看下如何读取XML的Bean配置并注册BeanDefinition到beanFactory。
如下：
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```

继续往下看：
进入AbstractRefreshableApplicationContext.refreshBeanFactory()方法，如下：
```java
if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
		}

```
可以看到先是销毁已有的BeanFactory,销毁创建的已有的Bean。跟踪destroyBeans()可以看到下面的销毁bean(单例)的代码：
```java
protected void clearSingletonCache() {
		synchronized (this.singletonObjects) {
			this.singletonObjects.clear();
			this.singletonFactories.clear();
			this.earlySingletonObjects.clear();
			this.registeredSingletons.clear();
			this.singletonsCurrentlyInDestruction = false;
		}
	}

```
现在继续看创建和初始化BeanFactory的代码：
```java
//创建BeanFactory
    DefaultListableBeanFactory beanFactory = createBeanFactory();
            //设置序列化ID
            beanFactory.setSerializationId(getId());
            //定制化BeanFactory，拓展点
			customizeBeanFactory(beanFactory);
            //加载和存储BeanDefinition
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
```
我们继续看loadBeanDefinitions方法，来到AbstractXmlApplicationContext,如下：
```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.创建Bean定义信息的读取器
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.配置XmlBeanDefinitionReader
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.初始化XmlBeanDefinitionReader
		initBeanDefinitionReader(beanDefinitionReader);
        //这里是加载BeanDefinition主要的入口
		loadBeanDefinitions(beanDefinitionReader);
	}

```
继续深入：
```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
            //读取配置路径下的bean定义信息
			reader.loadBeanDefinitions(configLocations);
		}
	}
```
继续跟踪loadBeanDefinitions方法，来到AbstractBeanDefinitionReader.loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources)方法：
```java
		ResourceLoader resourceLoader = getResourceLoader();
        if (resourceLoader instanceof ResourcePatternResolver resourcePatternResolver) {
        // Resource pattern matching available.

        Resource[] resources = resourcePatternResolver.getResources(location);
        int count = loadBeanDefinitions(resources);
        if (actualResources != null) {
        Collections.addAll(actualResources, resources);
        }
        return count;


        }
        else {
        // Can only load single resources by absolute URL.
        Resource resource = resourceLoader.getResource(location);
        int count = loadBeanDefinitions(resource);
        if (actualResources != null) {
        actualResources.add(resource);
        }
        return count;
        }
```
这里依旧还是读取资源，调用loadBeanDefinitions方法。继续跟踪方法，进入XmlBeanDefinitionReader的loadBeanDefinitions(Resource resource)方法：
```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
```
继续跟踪：
```java
public int loadBeanDefinitions(EncodedResource encodedResource) {
	
	try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
		InputSource inputSource = new InputSource(inputStream);
		if (encodedResource.getEncoding() != null) {
			inputSource.setEncoding(encodedResource.getEncoding());
		}
		return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
	}
	
	
}
```
最好来到了doLoadBeanDefinitions方法，看名字这个方法应该就是真正读取和注册Bean定义信息的方法了。
我们继续阅读doLoadBeanDefinitions方法：
```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			return count;
	}

```
这里就两个步骤，配置资源转换为XML的Document，如何从Document对象读取元素，解析为BeanDefinition，注册到BeanFactory里面。
doLoadDocument方法就不看了，继续跟踪registerBeanDefinitions方法：
```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```
我们继续跟踪registerBeanDefinitions(Document doc, XmlReaderContext readerContext)方法，来到DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions(Element root)方法，
这个方法以Do开头，看样子是真正干活的了。
```java
protected void doRegisterBeanDefinitions(Element root) {
	// Any nested <beans> elements will cause recursion in this method. In
	// order to propagate and preserve <beans> default-* attributes correctly,
	// keep track of the current (parent) delegate, which may be null. Create
	// the new (child) delegate with a reference to the parent for fallback purposes,
	// then ultimately reset this.delegate back to its original (parent) reference.
	// this behavior emulates a stack of delegates without actually necessitating one.
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);

	if (this.delegate.isDefaultNamespace(root)) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			// We cannot use Profiles.of(...) since profile expressions are not supported
			// in XML config. See SPR-12458 for details.
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return;
			}
		}
	}

	preProcessXml(root);
	parseBeanDefinitions(root, this.delegate);
	postProcessXml(root);

	this.delegate = parent;
}
```
看代码，读取Bean配置转交给了委派对象BeanDefinitionParserDelegate，我们继续向前：
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
这里可以看到，读取元素分为两类了，默认元素和客制化(定制化)元素。我们先进入默认元素的解析方法：
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```
好，现在终于看到&lt;bean>标签了。这里默认有四个标签import，alias，bean，beans。读取import就是前面的AbstractBeanDefinitionReader.lo
adBeanDefinitions(String location, @Nullable Set<Resource> actualResources)方法，nest是递归读取，这些不再解读了。注册alias别名是
最终到达了：	"this.aliasMap.put(alias, name);"，这里注册了bean的别名列表。
现在主要看下processBeanDefinition这个方法：
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    //从delegate读取BeanDefinition，并包装为修饰类
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
        //进行一修饰动作
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        //注册到BeanDefinitionRegistry/DefaultListableBeanFactory中
			// Register the final decorated instance.
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		// Send registration event.发布注册完成的事件通知
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```
我们现在跟踪parseBeanDefinitionElement方法：
```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
	String id = ele.getAttribute(ID_ATTRIBUTE);
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	List<String> aliases = new ArrayList<>();
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}
	String beanName = id;

	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
					beanName = this.readerContext.generateBeanName(beanDefinition);
					// Register an alias for the plain bean class name, if still possible,
					// if the generator returned the class name plus a suffix.
					// This is expected for Spring 1.2/2.0 backwards compatibility.
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName);
					}
				}
				
			}
			
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}

	return null;
}

```
我们看到这里主要读取了id/name，继续往下看
```java
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, @Nullable BeanDefinition containingBean) {

        this.parseState.push(new BeanEntry(beanName));

        String className = null;
        if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
        }
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }

        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        parseMetaElements(ele, bd);
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        parseConstructorArgElements(ele, bd);
        parsePropertyElements(ele, bd);
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;


        return null;
        }
```
这里读取Bean元素的各项属性，不再细化展开了。
说明一下：笔者这个系列的文章会对源码做一些删减，主要去掉非主要执行分支、异常、日志等处理，尽量关注于主要核心流程和核心代码。
现在一个bean元素转换为BeanDefinition对象了，接下来就是注入到容器里面去了。
我们转到这行代码：
```java
	BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
```
继续能看到：
```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}

```
这里注册了BeanDefinition自身和别名。注册BeanDefinition主要是存入beanDefinitionMap中，代码如下（有删减）：
```java

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {
    //判断Beanname对应的是否已存在
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
	if (existingDefinition != null) {
        //覆盖旧的配置，这里要特别注意一下，Spring注册的Bean默认是类型名字首字母小写，一个App出现相同类名注册为Spring的Bean的话，后者覆盖前者.
        //后与前是看Spring加载XML配置和注解配置中，哪一个定义出现的先后
		else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
        //存入beanDefinitionMap
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	else {
        //已经开始创建Bean实例时
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
			synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition);
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				removeManualSingletonName(beanName);
			}
		}
		else {
            //直接存入
            // Still in startup registration phase
			this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			removeManualSingletonName(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}

}
```
至此，一个&lt;bean>元素解析为BeanDefinition对象并注册到BeanFactory的流程完成了。可以看到整体流程还是偏流畅清晰的，没有太多的分支，代码复杂
度不算高。Spring源码对整个流程提供个多个抽象对象，互相组合使用。
下面从笔者的debug流程简单看一下BeanDefinitionHolder，和beanDefinitionMap：



# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)    <br>
