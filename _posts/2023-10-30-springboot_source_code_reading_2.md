---
layout: post
title: "SpringBoot源码阅读二:SpringBoot启动流程"
date: 2023-10-30
tags: [ springboot ]
comments: true
author: zouhuanli
---

本文是SpringBoot源码阅读计划的第二篇文章，本文主要介绍一下SpringBoot的启动流程。
```java
 public static void main(String[] args) {
        SpringApplication.run(MySpringBootApplication.class, args);
    }
```
主类启动代码就这样一行，本文将简单解读一下SpringApplication的创建流程和run方法。
本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。

# 一、SpringApplication的创建流程

跟踪run方法来到：
```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```
和
```java
public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
	}
```
进入SpringApplication的构造器方法。
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
                            //资源加载器
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
                            //加载源，这里就是MySpringBootApplication.class
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
                            //判断APP应用的类型，类型有；NONE-无需WEB的jar应用，SERVLET、REACTIVE-两个类型的Web应用
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
                            //创建初始化器
		this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
                            //设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
                            //应用主类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
我们主要分析这几个方法。

## 1.确定webApplicationType

确定应用类型的方法deduceFromClasspath很简单，如下：
```java
static WebApplicationType deduceFromClasspath() {
                            //尝试加载“org.springframework.web.reactive.DispatcherHandler”成功，而不加载DispatcherServlet和ServletContainer
        if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
        && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
        }
                            //加载DispatcherServlet和ServletContainer某一个不成功
        for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
        return WebApplicationType.NONE;
              }
            }
                            //加载DispatcherServlet和ServletContainer都成功
        return WebApplicationType.SERVLET;
        }
```
可以看到，不引入reactive栈时候，加载DispatcherServlet和ServletContainer都成功时候应用类型是WebApplicationType.SERVLET，否则是默认的jar应用。
可以通过SpringApplication#setWebApplicationType手动设置类型，引用了web模块也可以设置为NONE类型。

## 2.设置初始化器

<strong>初始化器是及其重要的</strong>,在后续的各流程中会使用到这些初始化器。初始化器简单说是SpringBoot内部创建的一系列组件，在各个流程使用。<br>
我们回顾一下[《DispatcherServlet的初始化流程》](https://zouhuanli.github.io/spring_source_code_reading_14/)这篇文章，其中的initStrategies方法：
```java
protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```
这里从DispatcherServlet.properties文件创建默认的九大组件。
````properties
# Logging Systems
org.springframework.boot.logging.LoggingSystemFactory=\
org.springframework.boot.logging.logback.LogbackLoggingSystem.Factory,\
org.springframework.boot.logging.log4j2.Log4J2LoggingSystem.Factory,\
org.springframework.boot.logging.java.JavaLoggingSystem.Factory

# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader

# ConfigData Location Resolvers
org.springframework.boot.context.config.ConfigDataLocationResolver=\
org.springframework.boot.context.config.ConfigTreeConfigDataLocationResolver,\
org.springframework.boot.context.config.StandardConfigDataLocationResolver

# ConfigData Loaders
org.springframework.boot.context.config.ConfigDataLoader=\
org.springframework.boot.context.config.ConfigTreeConfigDataLoader,\
org.springframework.boot.context.config.StandardConfigDataLoader

# Application Context Factories
org.springframework.boot.ApplicationContextFactory=\
org.springframework.boot.web.reactive.context.ReactiveWebServerApplicationContextFactory,\
org.springframework.boot.web.servlet.context.ServletWebServerApplicationContextFactory

# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers

# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.env.EnvironmentPostProcessorApplicationListener

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor,\
org.springframework.boot.env.RandomValuePropertySourceEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor

# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.context.config.ConfigDataNotFoundFailureAnalyzer,\
org.springframework.boot.context.properties.IncompatibleConfigurationFailureAnalyzer,\
org.springframework.boot.context.properties.NotConstructorBoundInjectionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanCurrentlyInCreationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanDefinitionOverrideFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanNotOfRequiredTypeFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindValidationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.UnboundConfigurationPropertyFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.MutuallyExclusiveConfigurationPropertiesFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoSuchMethodFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoUniqueBeanDefinitionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.PortInUseFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.ValidationExceptionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyNameFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyValueFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.PatternParseFailureAnalyzer,\
org.springframework.boot.liquibase.LiquibaseChangelogMissingFailureAnalyzer,\
org.springframework.boot.web.context.MissingWebServerFactoryBeanFailureAnalyzer,\
org.springframework.boot.web.embedded.tomcat.ConnectorStartFailureAnalyzer

# Failure Analysis Reporters
org.springframework.boot.diagnostics.FailureAnalysisReporter=\
org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter

# Database Initializer Detectors
org.springframework.boot.sql.init.dependency.DatabaseInitializerDetector=\
org.springframework.boot.flyway.FlywayDatabaseInitializerDetector,\
org.springframework.boot.jdbc.init.DataSourceScriptDatabaseInitializerDetector,\
org.springframework.boot.liquibase.LiquibaseDatabaseInitializerDetector,\
org.springframework.boot.orm.jpa.JpaDatabaseInitializerDetector,\
org.springframework.boot.r2dbc.init.R2dbcScriptDatabaseInitializerDetector

# Depends On Database Initialization Detectors
org.springframework.boot.sql.init.dependency.DependsOnDatabaseInitializationDetector=\
org.springframework.boot.sql.init.dependency.AnnotationDependsOnDatabaseInitializationDetector,\
org.springframework.boot.jdbc.SpringJdbcDependsOnDatabaseInitializationDetector,\
org.springframework.boot.jooq.JooqDependsOnDatabaseInitializationDetector,\
org.springframework.boot.orm.jpa.JpaDependsOnDatabaseInitializationDetector

````

这里创建初始化器/工厂类与其相似，从“META-INF/spring.factories”读取初始化器/工厂类的默认实现策略，在获取初始器时候进行实例化。<br>
我们跟踪getSpringFactoriesInstances方法来到：
```java
                        //FACTORIES_RESOURCE_LOCATION="META-INF/spring.factories"
public static SpringFactoriesLoader forDefaultResourceLocation(@Nullable ClassLoader classLoader) {
		return forResourceLocation(FACTORIES_RESOURCE_LOCATION, classLoader);
	}
```
这里加载了spring.factories的所有配置。继续进入这两个方法：
```java
public static SpringFactoriesLoader forResourceLocation(String resourceLocation, @Nullable ClassLoader classLoader) {
		Assert.hasText(resourceLocation, "'resourceLocation' must not be empty");
		ClassLoader resourceClassLoader = (classLoader != null ? classLoader :
				SpringFactoriesLoader.class.getClassLoader());
		Map<String, SpringFactoriesLoader> loaders = cache.computeIfAbsent(
				resourceClassLoader, key -> new ConcurrentReferenceHashMap<>());
		return loaders.computeIfAbsent(resourceLocation, key ->
				new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)));
	}

	protected static Map<String, List<String>> loadFactoriesResource(ClassLoader classLoader, String resourceLocation) {
		Map<String, List<String>> result = new LinkedHashMap<>();
		try {
			Enumeration<URL> urls = classLoader.getResources(resourceLocation);
			while (urls.hasMoreElements()) {
				UrlResource resource = new UrlResource(urls.nextElement());
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				properties.forEach((name, value) -> {
					String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String) value);
					List<String> implementations = result.computeIfAbsent(((String) name).trim(),
							key -> new ArrayList<>(factoryImplementationNames.length));
					Arrays.stream(factoryImplementationNames).map(String::trim).forEach(implementations::add);
				});
			}
			result.replaceAll(SpringFactoriesLoader::toDistinctUnmodifiableList);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" + resourceLocation + "]", ex);
		}
		return Collections.unmodifiableMap(result);
	}

```
其解析的配置的类型列表如下：

![factories](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-29-springboot_source_code_reading_2/factories.png)

和spring.factories配置信息一致,这里只是加载了spring.factories配置信息，并没有真正创建某一个工厂类的实例。<br>
创建工厂类的实例是通过load方法去懒加载来创建的，实际用到了哪个类型才创建，和Java SPI一样。<br>
load方法如下：
```java
                    //创建一个类型的,配置中的所有对应的实现类的实例
public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver,
			@Nullable FailureHandler failureHandler) {

		Assert.notNull(factoryType, "'factoryType' must not be null");
		List<String> implementationNames = loadFactoryNames(factoryType);
		logger.trace(LogMessage.format("Loaded [%s] names: %s", factoryType.getName(), implementationNames));
		List<T> result = new ArrayList<>(implementationNames.size());
		FailureHandler failureHandlerToUse = (failureHandler != null) ? failureHandler : THROWING_FAILURE_HANDLER;
		for (String implementationName : implementationNames) {
			T factory = instantiateFactory(implementationName, factoryType, argumentResolver, failureHandlerToUse);
			if (factory != null) {
				result.add(factory);
			}
		}
		AnnotationAwareOrderComparator.sort(result);
		return result;
	}
```
继续阅读是使用反射创建对应对象的，不再深入解读。<br>
而Java SPI是"Service Provider Interface"，是Java自带的服务拓展加载和发现机制，允许一个接口的实现类型从配置文件加载、解析，从而创建实现对象。<br>
例如，定义一个接口：
```java
/**
 * 一个正常的接口
 */
public interface TestSpiInterface {
    /**
     * 测试方法
     */
    void test();
}
```
以及三个实现类：
```java
/**
 * 实现类，可以是多个，可以不是本工程或者包，允许从第三方jar包中获取
 */
public class TestSpiInterfaceA implements TestSpiInterface {
    @Override
    public void test() {
        System.out.println("TestSpiInterfaceA");
    }
}


/**
 * 实现类，可以是多个，可以不是本工程或者包，允许从第三方jar包中获取
 */
public class TestSpiInterfaceB implements TestSpiInterface {
    @Override
    public void test() {
        System.out.println("TestSpiInterfaceB");
    }
}
/**
 * 实现类，可以是多个，可以不是本工程或者包，允许从第三方jar包中获取
 */
public class TestSpiInterfaceC implements TestSpiInterface {
    @Override
    public void test() {
        System.out.println("TestSpiInterfaceC");
    }
}
```
创建META-INF.services.com.homura.myspringboot.spi.TestSpiInterface文件，如下：
```text
com.homura.myspringboot.spi.impl.TestSpiInterfaceA
com.homura.myspringboot.spi.impl.TestSpiInterfaceC
```
测试主类如下：
```java
 public static void main(String[] args) {
        // 通过ServiceLoader加载实现类
        ServiceLoader<TestSpiInterface> serviceLoader = ServiceLoader.load(TestSpiInterface.class);
        for (TestSpiInterface testSpiInterface : serviceLoader) {
            testSpiInterface.test();
        }
    }
```
输出结果：
```text
TestSpiInterfaceA
TestSpiInterfaceC

```
SPI机制允许第三方实现拓展然后加入配置文件，增加了获得拓展的动态性。<br>
有的文章说到SpringFactoriesLoader也是SPI机制，这里不做过多比较和解读。

## 3.设置监听器

设置监听器和上面设置BootstrapRegistryInitializer、ApplicationContextInitializer是一样的，仅仅是加载拓展的类型不一样,也是SpringFactoriesLoader从spring.factories
加载ApplicationListener拓展的实现类型列表，要注意的是spring.factories整个配置列表只会加载一次，后续需要哪种拓展类型的实现类型直接进入load就行。<br>

deduceMainApplicationClass是确定主类，这里mainApplicationClass=MySpringBootApplication.class。<br>

现在SpringApplication创建完成了，接下来解读run方法，应用的启动方法。

# 二、SpringApplication#run方法

在解读run方法启动方法之前，我们简单回顾一下[<<AnnotationConfigApplicationContext初始化流程>>](https://zouhuanli.github.io/spring_source_code_reading_5/)和
[<<BeanFactoryPostProcessor解读>>](https://zouhuanli.github.io/spring_source_code_reading_20/)这两篇文章。<br>
在AnnotationConfigApplicationContext初始化流程这里，我们知道ApplicationContext首先注册了应用启动的配置类AppConfig这个Bean，然后在refresh流程的中的invokeBeanDefinitionRegistryPostProcessors使用<strong>ConfigurationClassPostProcessor</strong>去解析和注册注解配置的bean，期间用到了ConfigurationClassBeanDefinitionReader、AnnotatedBeanDefinitionReader、ConfigurationClassParser、ComponentScanAnnotationParser、ComponentScanBeanDefinitionParser等类去解析对应注解以及引入的注解配置的Bean。
因此，可以这里推断SpringBoot启动的ApplicationContext也是类似AnnotationConfigApplicationContext，也应是在invokeBeanFactoryPostProcessors方法内通过ConfigurationClassPostProcessor实现注解配置的Bean的注册流程。
此外因为WEB项目是默认创建内嵌的tomcat容器，run方法启动代码中也应该创建tomcat容器。
下面开始解读run方法。

## 1.创建BootstrapContext

创建启动上下文这个方法很简单，就是启动注册的bootstrapRegistryInitializers的初始化方法。
```java
private DefaultBootstrapContext createBootstrapContext() {
		DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
		this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
		return bootstrapContext;
	}
```
## 2.获得监听器和启动监听器
```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
		ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);
		argumentResolver = argumentResolver.and(String[].class, args);
		List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,
				argumentResolver);
		SpringApplicationHook hook = applicationHook.get();
		SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;
		if (hookListener != null) {
			listeners = new ArrayList<>(listeners);
			listeners.add(hookListener);
		}
		return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);
	}
```
这里的监听器依赖是从SpringFactoriesLoader读取到spring.factories去实例化,这里默认注册的监听器是EventPublishingRunListener。<br>
然后是调用监听器的starting方法：
```java
void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {
		doWithListeners("spring.boot.application.starting", (listener) -> listener.starting(bootstrapContext),
				(step) -> {
					if (mainApplicationClass != null) {
						step.tag("mainApplicationClass", mainApplicationClass.getName());
					}
				});
	}
```
## 3.准备环境

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
		// Create and configure the environment
                            //创建环境
		ConfigurableEnvironment environment = getOrCreateEnvironment();
                            //配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
                            //发布监听事件：环境已准备好
		listeners.environmentPrepared(bootstrapContext, environment);
		DefaultPropertiesPropertySource.moveToEnd(environment);
		Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
				"Environment prefix cannot be set via properties.");
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
			environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```
跟踪environmentPrepared方法，来到下面代码：
```java

	@Override
	public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
			ConfigurableEnvironment environment) {
		multicastInitialEvent(
				new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
	}
```
这里发布了ApplicationEnvironmentPreparedEvent这个事件，我们继续查找一下使用了这个事件的监听器，来到EnvironmentPostProcessorApplicationListener，来到下面方法:
```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationEnvironmentPreparedEvent environmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent(environmentPreparedEvent);
		}
		if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent();
		}
		if (event instanceof ApplicationFailedEvent) {
			onApplicationFailedEvent();
		}
	}
```
继续进入onApplicationEnvironmentPreparedEvent方法：
```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		SpringApplication application = event.getSpringApplication();
		for (EnvironmentPostProcessor postProcessor : getEnvironmentPostProcessors(application.getResourceLoader(),
				event.getBootstrapContext())) {
			postProcessor.postProcessEnvironment(environment, application);
		}
	}
```
在来到ConfigDataEnvironmentPostProcessor#postProcessEnvironment方法
```java
@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		postProcessEnvironment(environment, application.getResourceLoader(), application.getAdditionalProfiles());
	}
```
继续进入发现创建了ConfigDataEnvironment这个对象，这个类型是项目配置的类，其在下面代码设置了默认扫描配置文件的路径。
```java
static {
		List<ConfigDataLocation> locations = new ArrayList<>();
		locations.add(ConfigDataLocation.of("optional:classpath:/;optional:classpath:/config/"));
		locations.add(ConfigDataLocation.of("optional:file:./;optional:file:./config/;optional:file:./config/*/"));
		DEFAULT_SEARCH_LOCATIONS = locations.toArray(new ConfigDataLocation[0]);
	}
```
## 4.createApplicationContext创建容器和上下文

createApplicationContext方法只是按照应用类型创建ApplicationContext上下文对象。
在解读ApplicationContext整体结构的文章中有提及到主要有两类的上下文，一类是refreshable的可以刷新的，一类是generic不刷新的。而示例工程是一个Web项目因此创建的ApplicationContext应该是generic的子类，应该是基于注解的AnnotationWebXXXContext这样的。
跟踪测试流程发现这里创建的是AnnotationConfigServletWebServerApplicationContext(名字好长啊),其类关系图如下：

![AnnotationConfigServletWebServerApplicationContext](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-29-springboot_source_code_reading_2/AnnotationConfigServletWebServerApplicationContext.png)

可以看到这里boot对标准Spring的ApplicationContext做了一些拓展和定制化。而手动设置应用类型为NONE，普通的标准Java应用自然是上下文对象是AnnotationConfigApplicationContext。


## 5.prepareContext准备上下文

准备上下文是在context的刷新方法之前做一下前置处理。

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
                        //设置环境
		context.setEnvironment(environment);
                        //一些后置处理
		postProcessApplicationContext(context);
                        //aot相关
		addAotGeneratedInitializerIfNecessary(this.initializers);
                        //初始化器的初始化方法
		applyInitializers(context);
		listeners.contextPrepared(context);
		bootstrapContext.close(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
                        //注册相关Bean
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
			autowireCapableBeanFactory.setAllowCircularReferences(this.allowCircularReferences);
			if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
				listableBeanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
			}
		}
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
		if (!AotDetector.useGeneratedArtifacts()) {
			// Load the sources
			Set<Object> sources = getAllSources();
			Assert.notEmpty(sources, "Sources must not be empty");
                        //注意看这行代码，注册启动类，也就是MySpringBootApplication这个类，和AnnotationConfigApplicationContext注册启动主配置类一样。
			load(context, sources.toArray(new Object[0]));
		}
		listeners.contextLoaded(context);
	}
```
我们继续跟踪BeanDefinitionLoader#load方法，如下：
```java
private void load(Object source) {
		Assert.notNull(source, "Source must not be null");
		if (source instanceof Class<?> clazz) {
			load(clazz);
			return;
		}
		if (source instanceof Resource resource) {
			load(resource);
			return;
		}
		if (source instanceof Package pack) {
			load(pack);
			return;
		}
		if (source instanceof CharSequence sequence) {
			load(sequence);
			return;
		}
		throw new IllegalArgumentException("Invalid source type " + source.getClass());
	}
```
BeanDefinitionLoader是融合读取XML的bean配置，注解的Bean配置(Component、@Configuration、@Bean等)，以及groovy配置这三种的综合型的读取器。<br>
然后进入AnnotatedBeanDefinitionReader的注册方法，注册主配置类MySpringBootApplication到容器BeanFactory中，存入beanDefinitionMap。<br>
这里只注册了主配置类和一些框架本身的基础设施类，尚未没有开始注册应用的全部的bean ,如下。

![beanDefinitions](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-29-springboot_source_code_reading_2/beanDefinitions.png)

应用内其他的Bean是在refresh方法内部的invokeBeanFactoryPostProcessors方法通过ConfigurationClassPostProcessor去注册的。

## 6.refresh刷新容器
    todo~
### 6.1 特别点1：onRefresh方法

### 6.2 特别点2：invokeBeanFactoryPostProcessors方法

## 7.afterRefresh钩子方法

## 8.callRunners执行

## 9.started监听事件

## 10.执行runners

# 四、参考材料

1.《Spring Boot Reference Documentation》(version 3.1.5)<br>
2.SpringBoot源码(版本3.1.3)
