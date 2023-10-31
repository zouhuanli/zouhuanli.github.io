---
layout: post
title: "SpringBoot源码阅读三:自动配置实现原理"
date: 2023-10-31
tags: [ springboot ]
comments: true
author: zouhuanli
---

本文是SpringBoot源码阅读计划的第三篇文章，本文简单探究一下SpringBoot自动配置实现原理。

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。

如果说spring-boot-xxx-starter实现了某一个组件(功能)的自动依赖管理，通过gradle或maven依赖导入不仅仅指指定了该组件(功能)的依赖项，还指定了版本。
比如mybatis-spring-boot-starter组件
```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
    </dependency>
  </dependencies>
```
这里引入了mybatis、mybatis-spring依赖以及其版本号。
```xml
 <mybatis-spring.version>3.0.2</mybatis-spring.version>
```
而SpringBoot的自动配置是通过XXXAutoConfiguration类来实现的，比如DataSourceAutoConfiguration。而此类AutoConfiguration类是如何注册到容器，以及何处起作用的？<br>
这是本文需要解答的。

# 一、@SpringBootApplication注解解读

回到MySpringBootApplication这个启动类，如下。

```java
@SpringBootApplication
public class MySpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApplication.class, args);
    }

}
```

这里只有一个SpringBootApplication注解，我们进入这个注解。
```java
/**
 * Indicates a {@link Configuration configuration} class that declares one or more
 * {@link Bean @Bean} methods and also triggers {@link EnableAutoConfiguration
 * auto-configuration} and {@link ComponentScan component scanning}. This is a convenience
 * annotation that is equivalent to declaring {@code @SpringBootConfiguration},
 * {@code @EnableAutoConfiguration} and {@code @ComponentScan}.
 *
 * @author Phillip Webb
 * @author Stephane Nicoll
 * @author Andy Wilkinson
 * @since 1.2.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	/**
	 * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
	 * for a type-safe alternative to String-based package names.
	 * <p>
	 * <strong>Note:</strong> this setting is an alias for
	 * {@link ComponentScan @ComponentScan} only. It has no effect on {@code @Entity}
	 * scanning or Spring Data {@link Repository} scanning. For those you should add
	 * {@link org.springframework.boot.autoconfigure.domain.EntityScan @EntityScan} and
	 * {@code @Enable...Repositories} annotations.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * <p>
	 * <strong>Note:</strong> this setting is an alias for
	 * {@link ComponentScan @ComponentScan} only. It has no effect on {@code @Entity}
	 * scanning or Spring Data {@link Repository} scanning. For those you should add
	 * {@link org.springframework.boot.autoconfigure.domain.EntityScan @EntityScan} and
	 * {@code @Enable...Repositories} annotations.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

	/**
	 * The {@link BeanNameGenerator} class to be used for naming detected components
	 * within the Spring container.
	 * <p>
	 * The default value of the {@link BeanNameGenerator} interface itself indicates that
	 * the scanner used to process this {@code @SpringBootApplication} annotation should
	 * use its inherited bean name generator, e.g. the default
	 * {@link AnnotationBeanNameGenerator} or any custom instance supplied to the
	 * application context at bootstrap time.
	 * @return {@link BeanNameGenerator} to use
	 * @see SpringApplication#setBeanNameGenerator(BeanNameGenerator)
	 * @since 2.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
	Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

	/**
	 * Specify whether {@link Bean @Bean} methods should get proxied in order to enforce
	 * bean lifecycle behavior, e.g. to return shared singleton bean instances even in
	 * case of direct {@code @Bean} method calls in user code. This feature requires
	 * method interception, implemented through a runtime-generated CGLIB subclass which
	 * comes with limitations such as the configuration class and its methods not being
	 * allowed to declare {@code final}.
	 * <p>
	 * The default is {@code true}, allowing for 'inter-bean references' within the
	 * configuration class as well as for external calls to this configuration's
	 * {@code @Bean} methods, e.g. from another configuration class. If this is not needed
	 * since each of this particular configuration's {@code @Bean} methods is
	 * self-contained and designed as a plain factory method for container use, switch
	 * this flag to {@code false} in order to avoid CGLIB subclass processing.
	 * <p>
	 * Turning off bean method interception effectively processes {@code @Bean} methods
	 * individually like when declared on non-{@code @Configuration} classes, a.k.a.
	 * "@Bean Lite Mode" (see {@link Bean @Bean's javadoc}). It is therefore behaviorally
	 * equivalent to removing the {@code @Configuration} stereotype.
	 * @since 2.2
	 * @return whether to proxy {@code @Bean} methods
	 */
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;

}

```
可以看到主要设置了排除项和成员扫描路径。其使用的SpringBootConfiguration标识是一个SpringBoot的配置类，也是使用的@Configuration注解。<br>
@ComponentScan设置了扫描路径以及排除项。<br>
我们重点看一下EnableAutoConfiguration这个注解，见名知其义，这里是开启自动配置的注解。
```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    /**
     * Environment property that can be used to override when auto-configuration is
     * enabled.
     */
                        //是否启用自动配置的参数，可以设置值来覆盖是否启动自动配置
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    /**
     * Exclude specific auto-configuration classes such that they will never be applied.
     * @return the classes to exclude
     */
                        //排除项
    Class<?>[] exclude() default {};

    /**
     * Exclude specific auto-configuration class names such that they will never be
     * applied.
     * @return the class names to exclude
     * @since 1.3.0
     */
                        //排除项
    String[] excludeName() default {};

}

```
这里有两个注解。@AutoConfigurationPackage，应该是与扫描的包路径相关。AutoConfigurationImportSelector主要处理@EnableAutoConfiguration注解，去引入XXXAutoConfiguration等组件Bean。
通过@AutoConfigurationPackage注解进入AutoConfigurationPackages这个类。
```java
	@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
		}
```
和
```java
PackageImports(AnnotationMetadata metadata) {
			AnnotationAttributes attributes = AnnotationAttributes
				.fromMap(metadata.getAnnotationAttributes(AutoConfigurationPackage.class.getName(), false));
			List<String> packageNames = new ArrayList<>(Arrays.asList(attributes.getStringArray("basePackages")));
			for (Class<?> basePackageClass : attributes.getClassArray("basePackageClasses")) {
				packageNames.add(basePackageClass.getPackage().getName());
			}
			if (packageNames.isEmpty()) {
				packageNames.add(ClassUtils.getPackageName(metadata.getClassName()));
			}
			this.packageNames = Collections.unmodifiableList(packageNames);
		}

```
这里取得了默认的扫描组件的基本路径，示例工程是"com.homura.myspringboot',就是MySpringBootApplication的包路径。所以一般会把启动类放置到工程的顶层包里面。

下面重点解读一下AutoConfigurationImportSelector。

# 二、AutoConfigurationImportSelector解读

AutoConfigurationImportSelector是决定那些AutoConfiguration类型需要导入的核心管理类。<br>
先看下类关系图：

![AutoConfigurationImportSelector](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/AutoConfigurationImportSelector.png)

可以看到其顶层接口是Aware和ImportSelector，ImportSelector主要是选择哪些@Configuration配置类需要引入。<br>
我们进入AutoConfigurationImportSelector#selectImports方法，

```java
@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```

继续来到getAutoConfigurationEntry方法。

```java

	/**
	 * Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
	 * of the importing {@link Configuration @Configuration} class.
	 * @param annotationMetadata the annotation metadata of the configuration class
	 * @return the auto-configurations that should be imported
	 */
	protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
                                //注意这行代码，获得自动配置类的候选集合
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
                                //去重
		configurations = removeDuplicates(configurations);
                                //排除项
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
                                //过滤器
		configurations = getConfigurationClassFilter().filter(configurations);
                                //发布事件
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
```

我们继续进入getCandidateConfigurations方法。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
			.getCandidates();
		Assert.notEmpty(configurations,
				"No auto configuration classes found in "
						+ "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

再来到这个load方法：
```java
public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
		Assert.notNull(annotation, "'annotation' must not be null");
		ClassLoader classLoaderToUse = decideClassloader(classLoader);
                        //自动配置类的导入路径，这里是AutoConfiguration.class的全名+.imports后缀
                        //也就是org.springframework.boot.autoconfigure.AutoConfiguration.imports这个文件
		String location = String.format(LOCATION, annotation.getName());
		Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location);
		List<String> importCandidates = new ArrayList<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			importCandidates.addAll(readCandidateConfigurations(url));
		}
		return new ImportCandidates(importCandidates);
	}
```
这里导入了AopAutoConfiguration、PropertyPlaceholderAutoConfiguration等诸多AutoConfiguration类的类名。目前版本的Boot里面有148个自动配置类。
经过一些排除和过滤，示例工程AutoConfiguration类共有34个就全部导入了，如下。<br>

![configurations](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/configurations.png)

那么AutoConfiguration类是在那里注册到容器，和SpringFramework相结合呢？
我们从ConfigurationClassPostProcessor#processConfigBeanDefinitions方法一路追踪，来到ConfigurationClassParser#parse方法,再进入ConfigurationClassParser#doProcessConfigurationClass方法，
最后来到processImports方法。

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports) {

		if (importCandidates.isEmpty()) {
			return;
		}

		if (checkForCircularImports && isChainedImportOnStack(configClass)) {
			this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
		}
		else {
			this.importStack.push(configClass);
			try {
				for (SourceClass candidate : importCandidates) {
                    //注意看这个分支
					if (candidate.isAssignable(ImportSelector.class)) {
						// Candidate class is an ImportSelector -> delegate to it to determine imports
						Class<?> candidateClass = candidate.loadClass();
						ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
								this.environment, this.resourceLoader, this.registry);
						Predicate<String> selectorFilter = selector.getExclusionFilter();
						if (selectorFilter != null) {
							exclusionFilter = exclusionFilter.or(selectorFilter);
						}
						if (selector instanceof DeferredImportSelector deferredImportSelector) {
							this.deferredImportSelectorHandler.handle(configClass, deferredImportSelector);
						}
						else {
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
							processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
						}
					}
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
						// Candidate class is an ImportBeanDefinitionRegistrar ->
						// delegate to it to register additional bean definitions
						Class<?> candidateClass = candidate.loadClass();
						ImportBeanDefinitionRegistrar registrar =
								ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
										this.environment, this.resourceLoader, this.registry);
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					else {
						// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
						// process it as an @Configuration class
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
						processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
					}
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]: " + ex.getMessage(), ex);
			}
			finally {
				this.importStack.pop();
			}
		}
	}
```
跟踪执行流程，如下:

![importCandidates](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/importCandidates.png)

这里创建AutoConfigurationImportSelector对象,并加入deferredImportSelectorHandler。
然后来到:

```java
this.deferredImportSelectorHandler.process();
```

进入process方法：

```java
public void process() {
			List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
			this.deferredImportSelectors = null;
			try {
				if (deferredImports != null) {
					DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
					deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
					deferredImports.forEach(handler::register);
					handler.processGroupImports();
				}
			}
			finally {
				this.deferredImportSelectors = new ArrayList<>();
			}
		}
```
再来到processGroupImports这个方法，按照组导入,这里的默认组是AutoConfigurationGroup。
```java
public void processGroupImports() {
			for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
				Predicate<String> exclusionFilter = grouping.getCandidateFilter();
				grouping.getImports().forEach(entry -> {
					ConfigurationClass configurationClass = this.configurationClasses.get(entry.getMetadata());
					try {
                                    //这里导入和注册AutoConfiguration类
						processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
								Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)),
								exclusionFilter, false);
					}
					catch (BeanDefinitionStoreException ex) {
						throw ex;
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to process import candidates for configuration class [" +
										configurationClass.getMetadata().getClassName() + "]", ex);
					}
				});
			}
		}
```
这里”grouping.getImports()“并不是上面.imports里面全部的，和上面解读的34个自动配置类一样。

![grouping_imports](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/grouping_imports.png)

这里processImports方法就是处理一个具体的XXXAutoConfiguration类的相关注册信息。
```java
processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
								Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)),
								exclusionFilter, false)
```
这里第三个参数importCandidates就是一个具体的AutoConfiguration类了，如org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration。

到这里AutoConfiguration自动配置类及其引入的Bean配置就注册到容器，下面具体解读一个AutoConfiguration自动配置类解读自动配置的执行流程。

# 三、AutoConfiguration解读

todo~

# 四、参考材料

1.《Spring Boot Reference Documentation》(version 3.1.5)<br>
2.SpringBoot源码(版本3.1.3)
