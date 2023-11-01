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

那么AutoConfiguration类是在哪里注册到容器，和SpringFramework相结合呢？
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

这里以DataSourceAutoConfiguration这个自动配置类作为示例。<br>
解读DataSourceAutoConfiguration之前，考虑一个及其简单的数据源自动配置类，全部使用Java代码+注解实现，可以简单的这样做。<br>
先配置一个PropertySourcesPlaceholderConfigurer实例。

```java
@Configuration
public class PropertySourcesPlaceholderConfigurerConfig {

    @Bean
    public PropertySourcesPlaceholderConfigurer propertyConfigurer() {
        PropertySourcesPlaceholderConfigurer propertyConfigurer = new PropertySourcesPlaceholderConfigurer();
        propertyConfigurer.setIgnoreUnresolvablePlaceholders(true);
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        Resource classPathResource = resolver.getResource("classpath:springmvc-db.properties");
        propertyConfigurer.setLocation(classPathResource);
        return propertyConfigurer;
    }
```

再配置数据库连接配置的PropertiesBean。

```java
@Data
@Configuration
public class DataSourcePropertiesBean {
    /**
     * <property name="driverClassName" value="${db.driver}"/>
     * <property name="jdbcUrl" value="${db.url}"/>
     * <property name="username" value="${db.username}"/>
     * <property name="password" value="${db.password}"/>
     * <property name="connectionTestQuery" value="${db.connectionTestQuery}"/>
     * <!-- 生效超时 -->
     * <property name="validationTimeout" value="${db.validationTimeout}"/>
     * <!-- 连接只读数据库时配置为true， 保证安全 -->
     * <property name="readOnly" value="${db.readOnly}"/>
     * <!-- 等待连接池分配连接的最大时长（毫秒），超过这个时长还没可用的连接则发生SQLException， 缺省:30秒 -->
     * <property name="connectionTimeout" value="${db.connectionTimeout}"/>
     * <!-- 一个连接idle状态的最大时长（毫秒），超时则被释放（retired），缺省:10分钟 -->
     * <property name="idleTimeout" value="${db.idleTimeout}"/>
     * <!-- 一个连接的生命时长（毫秒），超时而且没被使用则被释放（retired），缺省:30分钟，建议设置比数据库超时时长少30秒，参考MySQL
     * wait_timeout参数（show variables like '%timeout%';） -->
     * <property name="maxLifetime" value="${db.maxLifetime}"/>
     * <!-- 连接池中允许的最大连接数。缺省值：10；推荐的公式：((core_count * 2) + effective_spindle_count) -->
     * <property name="maximumPoolSize" value="${db.maximumPoolSize}"/>
     **/
    @Value("${db.driver}")
    private String driverClassName;
    @Value("${db.url}")
    private String jdbcUrl;
    @Value("${db.username}")
    private String username;
    @Value("${db.password}")
    private String password;
    @Value("${db.connectionTestQuery}")
    private String connectionTestQuery;
    @Value("${db.validationTimeout}")
    private long validationTimeout;
    @Value("${db.readOnly}")
    private boolean readOnly;
    @Value("${db.connectionTimeout}")
    private long connectionTimeout;
    @Value("${db.idleTimeout}")
    private long idleTimeout;
    @Value("${db.maxLifetime}")
    private long maxLifetime;
    @Value("${db.maximumPoolSize}")
    private int maximumPoolSize;
    
}
```

再创建数据源Datasource。
```java
@Configuration
public class DataSourceConfiguration {
    @Bean(name = "dataSource")
    public HikariDataSource dataSource(@Autowired @Qualifier("dataSourcePropertiesBean") DataSourcePropertiesBean dataSourceConfig) {
        return buildDataSource(dataSourceConfig);
    }
    private HikariDataSource buildDataSource(DataSourcePropertiesBean dataSourceConfig) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setPoolName("HikariDataSource-Pool-1");
        dataSource.setDriverClassName(dataSourceConfig.getDriverClassName());
        dataSource.setJdbcUrl(dataSourceConfig.getJdbcUrl());
        dataSource.setUsername(dataSourceConfig.getUsername());
        dataSource.setPassword(dataSourceConfig.getPassword());
        dataSource.setConnectionTestQuery(dataSourceConfig.getConnectionTestQuery());
        dataSource.setValidationTimeout(dataSourceConfig.getValidationTimeout());
        dataSource.setReadOnly(dataSourceConfig.isReadOnly());
        dataSource.setConnectionTimeout(dataSourceConfig.getConnectionTimeout());
        dataSource.setIdleTimeout(dataSourceConfig.getIdleTimeout());
        dataSource.setMaxLifetime(dataSourceConfig.getMaxLifetime());
        dataSource.setMaximumPoolSize(dataSourceConfig.getMaximumPoolSize());

        return dataSource;
    }
```
这样name为dataSource，类型为HikariDataSource的数据源对象就创建好了。这里使用到了PropertySourcesPlaceholderConfigurer这个对象来解析@value注入的配置项。<br>
这只是非常简陋的实现方法，不过也可以帮助我们理解DataSourceAutoConfiguration这个自动配置类的实现原理。<br>
因此，可以作一下初步的假设DataSourceAutoConfiguration这个配置类首先要解析数据库配置，生产XXXPropertiesBean这样一个对象来保存配置，然后创建Datasource的Bean对象。<br>
我们开始看下这个类的源码。

## 1.DataSourceAutoConfiguration

DataSourceAutoConfiguration源码如下：
```java
                                //AutoConfiguration注解标识是SpringBoot的自动配置类，注解里面使用了@Configuration
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
                                 //EnableConfigurationProperties注解，导入解析和存储配置项的Bean，这个注解引入了EnableConfigurationPropertiesRegistrar这个注册器
@EnableConfigurationProperties(DataSourceProperties.class)
                                //引入DataSourcePoolMetadataProvidersConfiguration，数据库连接池配置类
@Import(DataSourcePoolMetadataProvidersConfiguration.class)
public class DataSourceAutoConfiguration {
                                 //引入内嵌的数据源Datasource
	@Configuration(proxyBeanMethods = false)
	@Conditional(EmbeddedDatabaseCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import(EmbeddedDataSourceConfiguration.class)
	protected static class EmbeddedDatabaseConfiguration {

	}
                                 //创建连接池类型的数据源，有Hikari(默认)、Tomcat、Dbcp2、OracleUcp、Generic等类型
	@Configuration(proxyBeanMethods = false)
	@Conditional(PooledDataSourceCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
			DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
			DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
	protected static class PooledDataSourceConfiguration {
                                //数据库连接信息，看源码内部包含DataSourceProperties数据源配置信息这个对象。
		@Bean
		@ConditionalOnMissingBean(JdbcConnectionDetails.class)
		PropertiesJdbcConnectionDetails jdbcConnectionDetails(DataSourceProperties properties) {
			return new PropertiesJdbcConnectionDetails(properties);
		}

	}
}
```
我们进入这个类DataSourceConfiguration.Hikari。
```java
   @Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		static HikariJdbcConnectionDetailsBeanPostProcessor jdbcConnectionDetailsHikariBeanPostProcessor(
				ObjectProvider<JdbcConnectionDetails> connectionDetailsProvider) {
			return new HikariJdbcConnectionDetailsBeanPostProcessor(connectionDetailsProvider);
		}
                                     //重点代码，创建数据源对象
                                     //这里创建Datasource数据源对象依赖DataSourceProperties和JdbcConnectionDetails对象。
                                        //PropertiesJdbcConnectionDetails是DataSourceProperties的适配对象，不作过多解读。
                                        //重点解读一下DataSourceProperties这个类
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties, JdbcConnectionDetails connectionDetails) {
			HikariDataSource dataSource = createDataSource(connectionDetails, HikariDataSource.class,
					properties.getClassLoader());
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
```
接下来重点解读一下DataSourceProperties这个类。

## 2.DataSourceProperties

### 2.1 DataSourceProperties的注册和实例化

DataSourceProperties是解析和存储数据库配置的Bean，看属性和功能应和上面示例的DataSourcePropertiesBean一样。接下来要重点关注：<br>
 1.DataSourceProperties如何创建和何处创建？---注解Bean和创建Bean。<br>
 2.DataSourceProperties内部的各项属性(字段)如何获取到配置项文件的配置值？----属性绑定。<br>

相对于PropertiesJdbcConnectionDetails由应用显式创建(如下)，DataSourceProperties这个对象并未找到显式创建的代码。<br>
因此这里可以推断这个Bean应该首先在invokeBeanFactoryPostProcessors这里通过某一个XXXRegistrar完成BeanDefinition的注册，注册Bean信息到容器的beanDefinitionMap里面。然后finishBeanFactoryInitialization完成实例化真正创建这个对象。

```java
                        //显式创建Bean
        @Bean
		@ConditionalOnMissingBean(JdbcConnectionDetails.class)
		PropertiesJdbcConnectionDetails jdbcConnectionDetails(DataSourceProperties properties) {
			return new PropertiesJdbcConnectionDetails(properties);
		}
```

我们看下DataSourceProperties这个类。
```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {

    private ClassLoader classLoader;

    /**
     * Whether to generate a random datasource name.
     */
    private boolean generateUniqueName = true;

    /**
     * Datasource name to use if "generate-unique-name" is false. Defaults to "testdb"
     * when using an embedded database, otherwise null.
     */
    private String name;

    /**
     * Fully qualified name of the connection pool implementation to use. By default, it
     * is auto-detected from the classpath.
     */
    private Class<? extends DataSource> type;
    //......
}
```
这里定义了各项配置项如name、driver-class-name、url、username、password，这些配置项需要使用'spring.datasource'前缀。<br>
然后重点看一下@ConfigurationProperties这个注解:
```java
/**
 * Annotation for externalized configuration. Add this to a class definition or a
 * {@code @Bean} method in a {@code @Configuration} class if you want to bind and validate
 * some external Properties (e.g. from a .properties file).
 * <p>
 * Binding is either performed by calling setters on the annotated class or, if
 * {@link ConstructorBinding @ConstructorBinding} is in use, by binding to the constructor
 * parameters.
 * <p>
 * Note that contrary to {@code @Value}, SpEL expressions are not evaluated since property
 * values are externalized.
 *
 * @author Dave Syer
 * @since 1.0.0
 * @see ConfigurationPropertiesScan
 * @see ConstructorBinding
 * @see ConfigurationPropertiesBindingPostProcessor
 * @see EnableConfigurationProperties
 */
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface ConfigurationProperties {

	/**
	 * The prefix of the properties that are valid to bind to this object. Synonym for
	 * {@link #prefix()}. A valid prefix is defined by one or more words separated with
	 * dots (e.g. {@code "acme.system.feature"}).
	 * @return the prefix of the properties to bind
	 */
	@AliasFor("prefix")
	String value() default "";

	/**
	 * The prefix of the properties that are valid to bind to this object. Synonym for
	 * {@link #value()}. A valid prefix is defined by one or more words separated with
	 * dots (e.g. {@code "acme.system.feature"}).
	 * @return the prefix of the properties to bind
	 */
	@AliasFor("value")
	String prefix() default "";

	/**
	 * Flag to indicate that when binding to this object invalid fields should be ignored.
	 * Invalid means invalid according to the binder that is used, and usually this means
	 * fields of the wrong type (or that cannot be coerced into the correct type).
	 * @return the flag value (default false)
	 */
	boolean ignoreInvalidFields() default false;

	/**
	 * Flag to indicate that when binding to this object unknown fields should be ignored.
	 * An unknown field could be a sign of a mistake in the Properties.
	 * @return the flag value (default true)
	 */
	boolean ignoreUnknownFields() default true;

}

```
这个注解标识一个类型是自动配置类的读取配置项属性的PropertiesBean。<br>
进入EnableConfigurationProperties注解，来到EnableConfigurationPropertiesRegistrar这个类。<br>
EnableConfigurationPropertiesRegistrar父类是ImportBeanDefinitionRegistrar，用来引入注册Bean的信息，就是额外添加BeanDefinition信息到容器中。类似的有AspectJAutoProxyRegistrar，MapperScannerRegistrar(来自mybatis)等。

```java
class EnableConfigurationPropertiesRegistrar implements ImportBeanDefinitionRegistrar {

	private static final String METHOD_VALIDATION_EXCLUDE_FILTER_BEAN_NAME = Conventions
		.getQualifiedAttributeName(EnableConfigurationPropertiesRegistrar.class, "methodValidationExcludeFilter");

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		registerInfrastructureBeans(registry);
		registerMethodValidationExcludeFilter(registry);
		ConfigurationPropertiesBeanRegistrar beanRegistrar = new ConfigurationPropertiesBeanRegistrar(registry);
		getTypes(metadata).forEach(beanRegistrar::register);
	}

	private Set<Class<?>> getTypes(AnnotationMetadata metadata) {
		return metadata.getAnnotations()
			.stream(EnableConfigurationProperties.class)
			.flatMap((annotation) -> Arrays.stream(annotation.getClassArray(MergedAnnotation.VALUE)))
			.filter((type) -> void.class != type)
			.collect(Collectors.toSet());
	}

	static void registerInfrastructureBeans(BeanDefinitionRegistry registry) {
                                    //注册ConfigurationPropertiesBindingPostProcessor和ConfigurationPropertiesBinderFactory，只会注册一次
		ConfigurationPropertiesBindingPostProcessor.register(registry);
                                    //BoundConfigurationProperties,只会注册一次
		BoundConfigurationProperties.register(registry);
	}

	static void registerMethodValidationExcludeFilter(BeanDefinitionRegistry registry) {
		if (!registry.containsBeanDefinition(METHOD_VALIDATION_EXCLUDE_FILTER_BEAN_NAME)) {
			BeanDefinition definition = BeanDefinitionBuilder
				.rootBeanDefinition(MethodValidationExcludeFilter.class, "byAnnotation")
				.addConstructorArgValue(ConfigurationProperties.class)
				.setRole(BeanDefinition.ROLE_INFRASTRUCTURE)
				.getBeanDefinition();
			registry.registerBeanDefinition(METHOD_VALIDATION_EXCLUDE_FILTER_BEAN_NAME, definition);
		}
	}

}

```

跟踪这个类处理DatasourceAutoConfiguration的过程。

![EnableConfigurationPropertiesRegistrar](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/EnableConfigurationPropertiesRegistrar.png)

继续来到ConfigurationPropertiesBeanRegistrar这个类,这里注册了DataSourceProperties到容器中，后续容器会自动创建这个Bean对象。

```java
void register(Class<?> type) {
		MergedAnnotation<ConfigurationProperties> annotation = MergedAnnotations
			.from(type, SearchStrategy.TYPE_HIERARCHY)
			.get(ConfigurationProperties.class);
		register(type, annotation);
	}

	void register(Class<?> type, MergedAnnotation<ConfigurationProperties> annotation) {
		String name = getName(type, annotation);
		if (!containsBeanDefinition(name)) {
			registerBeanDefinition(name, type, annotation);
		}
	}
```
执行流程如图：
![ConfigurationPropertiesBeanRegistrar](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/ConfigurationPropertiesBeanRegistrar.png)

到这里，我们知道了：被@ConfigurationProperties注解修饰的DataSourceProperties已经注册到容器了，并由容器去实例化。其他@ConfigurationProperties注解修饰的XXXProperties也是同理。

既然DataSourceProperties的注册和实例化(finishBeanFactoryInitialization方法内)已经完成了，接下来要去解答配置文件的配置项如何注入到DataSourceProperties字段中，也就是DataSourceProperties的每个属性如何<strong>绑定</strong>配置项文件的配置。

### 2.2 配置文件属性解析和绑定

属性绑定简单说就对DataSourceProperties的字段赋值。
```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {

    private ClassLoader classLoader;

    /**
     * Whether to generate a random datasource name.
     */
    private boolean generateUniqueName = true;

    /**
     * Datasource name to use if "generate-unique-name" is false. Defaults to "testdb"
     * when using an embedded database, otherwise null.
     */
    private String name;

    /**
     * Fully qualified name of the connection pool implementation to use. By default, it
     * is auto-detected from the classpath.
     */
    private Class<? extends DataSource> type;

    /**
     * Fully qualified name of the JDBC driver. Auto-detected based on the URL by default.
     */
    private String driverClassName;

    /**
     * JDBC URL of the database.
     */
    private String url;
                    //......
}
```
比如说如何设置url的值。
EnableConfigurationPropertiesRegistrar中注册了ConfigurationPropertiesBindingPostProcessor和ConfigurationPropertiesBinder这两个组件，看名字是与属性绑定相关的？<br>
首先看下ConfigurationPropertiesBindingPostProcessor：
```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!hasBoundValueObject(beanName)) {
			bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
		}
		return bean;
	}
```
之前的文件解读过BeanPostProcessor#postProcessBeforeInitialization在Bean已经实例化，尚未初始化的时候执行。参考这篇文章[Bean的创建过程和Bean的生命周期](https://zouhuanli.github.io/spring_source_code_reading_6/)。
其在Spring源码中调用的位置是AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization方法。<br>
因此创建DataSourceProperties这个Bean已经实例化，尚未初始化的时候，会执行下面代码：
```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!hasBoundValueObject(beanName)) {
			bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
		}
		return bean;
	}
    
	private void bind(ConfigurationPropertiesBean bean) {
		if (bean == null) {
			return;
		}
		Assert.state(bean.asBindTarget().getBindMethod() != BindMethod.VALUE_OBJECT,
				"Cannot bind @ConfigurationProperties for bean '" + bean.getName()
						+ "'. Ensure that @ConstructorBinding has not been applied to regular bean");
		try {
			this.binder.bind(bean);
		}
		catch (Exception ex) {
			throw new ConfigurationPropertiesBindException(bean, ex);
		}
	}
```
然后我们来到ConfigurationPropertiesBinder。
```java
BindResult<?> bind(ConfigurationPropertiesBean propertiesBean) {
		Bindable<?> target = propertiesBean.asBindTarget();
		ConfigurationProperties annotation = propertiesBean.getAnnotation();
		BindHandler bindHandler = getBindHandler(target, annotation);
		return getBinder().bind(annotation.prefix(), target, bindHandler);
	}
```
这里是实现PropertiesBean的属性绑定的功能的代码了。<br>

![ConfigurationPropertiesBinder](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/ConfigurationPropertiesBinder.png)

我们跟踪BindHandler#bind方法来到这个方法。
```java
private Object bindDataObject(ConfigurationPropertyName name, Bindable<?> target, BindHandler handler,
			Context context, boolean allowRecursiveBinding) {
		if (isUnbindableBean(name, target, context)) {
			return null;
		}
		Class<?> type = target.getType().resolve(Object.class);
		BindMethod bindMethod = target.getBindMethod();
		if (!allowRecursiveBinding && context.isBindingDataObject(type)) {
			return null;
		}
		DataObjectPropertyBinder propertyBinder = (propertyName, propertyTarget) -> bind(name.append(propertyName),
				propertyTarget, handler, context, false, false);
		return context.withDataObject(type, () -> {
			for (DataObjectBinder dataObjectBinder : this.dataObjectBinders.get(bindMethod)) {
                                        //属性绑定
				Object instance = dataObjectBinder.bind(name, target, context, propertyBinder);
				if (instance != null) {
					return instance;
				}
			}
			return null;
		});
	}

```
继续来到JavaBeanBinder#bind方法。
```java
private <T> boolean bind(DataObjectPropertyBinder propertyBinder, Bean<T> bean, BeanSupplier<T> beanSupplier,
			Context context) {
		boolean bound = false;
                                    //这里遍历属性集合，处理每一个属性
                                     //这里就是[generate-unique-name, class-loader, driver-class-name, embedded-database-connection, jndi-name, name, password, type, url, username, xa, bean-class-loader]
                                    //对应DatasourceProperties的每个字段
		for (BeanProperty beanProperty : bean.getProperties().values()) {
			bound |= bind(beanSupplier, propertyBinder, beanProperty);
			context.clearConfigurationProperty();
		}
		return bound;
```

这里就是具体绑定一项属性了,如spring.datasource.password。
```java
private <T> Object bindObject(ConfigurationPropertyName name, Bindable<T> target, BindHandler handler,
			Context context, boolean allowRecursiveBinding) {
                        //查找配置的值
		ConfigurationProperty property = findProperty(name, target, context);
		if (property == null && context.depth != 0 && containsNoDescendantOf(context.getSources(), name)) {
			return null;
		}
		AggregateBinder<?> aggregateBinder = getAggregateBinder(target, context);
		if (aggregateBinder != null) {
			return bindAggregate(name, target, handler, context, aggregateBinder);
		}
		if (property != null) {
			try {
                        //绑定和注入
				return bindProperty(target, context, property);
			}
			catch (ConverterNotFoundException ex) {
				// We might still be able to bind it using the recursive binders
				Object instance = bindDataObject(name, target, handler, context, allowRecursiveBinding);
				if (instance != null) {
					return instance;
				}
				throw ex;
			}
		}
		return bindDataObject(name, target, handler, context, allowRecursiveBinding);
	}
```

![bindObject](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-31-springboot_source_code_reading_3/bindObject.png)

最后在：
```java
if (property.isSettable()) {
			property.setValue(beanSupplier, bound);
		}
```
使用反射调用setter来设置DatasourceProperties的对应字段的值。<br>

到这里DatasourceProperties已经创建成功，并且读取了对应的配置的值。

### 2.3 创建Datasource

回到DataSourceConfiguration$Hikari这个配置类。
```java
@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties, JdbcConnectionDetails connectionDetails) {
			HikariDataSource dataSource = createDataSource(connectionDetails, HikariDataSource.class,
					properties.getClassLoader());
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}
```
上面的解读过程中DataSourceProperties、JdbcConnectionDetails已经创建完成了，现在可以成功创建HikariDataSource了。<br>

要重点理解DataSourceProperties的注册和实例化(创建对象)、<strong>属性绑定(解析和存储配置文件的配置值)</strong>这两个流程,整体流程建议按照文章的解读内容结合实际调试boot源码来加深理解。<br>
本文从@SpringBootApplication注解开始，解读了自动配置的相关组件类，和某一个自动配置类的实现原理，由表到里，由浅入深，文章内容较多还有一些和SpringFramework源码解读相结合的内容，希望读者能够
按照流程去实际的SpringBoot代码，加深对自动配置实现原理的理解。
# 四、参考材料

1.《Spring Boot Reference Documentation》(version 3.1.5)<br>
2.SpringBoot源码(版本3.1.3)
