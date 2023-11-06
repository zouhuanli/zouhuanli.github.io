---
layout: post
title: "SpringBoot源码阅读六:Jar启动流程和部分SpringBoot注解解读"
date: 2023-11-03
tags: [ springboot ]
comments: true
author: zouhuanli
---

本文是SpringBoot源码阅读计划的第六篇文章，Jar启动过程和@Import、@Conditional注解。<br>
本篇文章算是对[<<自动配置实现原理>>](https://zouhuanli.github.io/springboot_source_code_reading_3/)的一个补充。
本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

# 一、Jar启动流程
在示例工程中,主类是：

```java
@SpringBootApplication
@ServletComponentScan(basePackages = {"com.homura.myspringboot"}) //
public class MySpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApplication.class, args);
    }

}

```
我们直接执行“java -jar MySpringBoot\MySpringBoot-0.0.1-SNAPSHOT.jar”,其运行主类并不是MySpringBootApplication，而是JarLauncher这个启动类。<br>
SpringBoot的jar包和普通的jar包有所不同。官方文档给的示例是：
```text
example.jar
|
+-META-INF
    | +-MANIFEST.MF
+-org
    | +-springframework
        | +-boot
            | +-loader
                | +-<spring boot loader classes>
+-BOOT-INF
    +-classes
        | +-mycompany
            | +-project
                | +-YourClasses.class
    +-lib
        +-dependency1.jar
        +-dependency2.jar
```
Application class 应放在嵌套的 BOOT-INF/classes 目录中。 依赖应该放在嵌套的 BOOTINF/lib 目录中。启动时先去/META-INF下MANIFEST.MF找到主类“org.springframework.boot.loader.JarLauncher”。
```text
Manifest-Version: 1.0
Created-By: Maven JAR Plugin 3.3.0
Build-Jdk-Spec: 17
Implementation-Title: MySpringBoot
Implementation-Version: 0.0.1-SNAPSHOT
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.homura.myspringboot.MySpringBootApplication
Spring-Boot-Version: 3.1.3
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Spring-Boot-Layers-Index: BOOT-INF/layers.idx


```
Classpath的索引文件在“ BOOT-INF/classpath.idx”。
```text
- "BOOT-INF/lib/spring-boot-3.1.3.jar"
- "BOOT-INF/lib/spring-boot-autoconfigure-3.1.3.jar"
- "BOOT-INF/lib/jakarta.annotation-api-2.1.1.jar"
- "BOOT-INF/lib/snakeyaml-1.33.jar"
- "BOOT-INF/lib/jackson-databind-2.15.2.jar"
- "BOOT-INF/lib/jackson-annotations-2.15.2.jar"
- "BOOT-INF/lib/jackson-core-2.15.2.jar"
- "BOOT-INF/lib/jackson-datatype-jdk8-2.15.2.jar"
- "BOOT-INF/lib/jackson-datatype-jsr310-2.15.2.jar"
- "BOOT-INF/lib/jackson-module-parameter-names-2.15.2.jar"
- "BOOT-INF/lib/tomcat-embed-core-10.1.12.jar"
- "BOOT-INF/lib/tomcat-embed-websocket-10.1.12.jar"
- "BOOT-INF/lib/spring-web-6.0.11.jar"
- "BOOT-INF/lib/spring-beans-6.0.11.jar"
- "BOOT-INF/lib/micrometer-observation-1.11.3.jar"
- "BOOT-INF/lib/micrometer-commons-1.11.3.jar"
- "BOOT-INF/lib/spring-webmvc-6.0.11.jar"
- "BOOT-INF/lib/spring-aop-6.0.11.jar"
- "BOOT-INF/lib/spring-context-6.0.11.jar"
- "BOOT-INF/lib/spring-expression-6.0.11.jar"
- "BOOT-INF/lib/log4j-slf4j2-impl-2.20.0.jar"
- "BOOT-INF/lib/log4j-api-2.20.0.jar"
- "BOOT-INF/lib/slf4j-api-2.0.7.jar"
- "BOOT-INF/lib/log4j-core-2.20.0.jar"
- "BOOT-INF/lib/log4j-jul-2.20.0.jar"
- "BOOT-INF/lib/spring-core-6.0.11.jar"
- "BOOT-INF/lib/spring-jcl-6.0.11.jar"
- "BOOT-INF/lib/guava-32.1.2-jre.jar"
- "BOOT-INF/lib/failureaccess-1.0.1.jar"
- "BOOT-INF/lib/listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar"
- "BOOT-INF/lib/jsr305-3.0.2.jar"
- "BOOT-INF/lib/checker-qual-3.33.0.jar"
- "BOOT-INF/lib/error_prone_annotations-2.18.0.jar"
- "BOOT-INF/lib/j2objc-annotations-2.8.jar"
- "BOOT-INF/lib/tomcat-embed-el-10.1.12.jar"
- "BOOT-INF/lib/hibernate-validator-8.0.1.Final.jar"
- "BOOT-INF/lib/jakarta.validation-api-3.0.2.jar"
- "BOOT-INF/lib/jboss-logging-3.5.3.Final.jar"
- "BOOT-INF/lib/classmate-1.5.1.jar"
- "BOOT-INF/lib/mybatis-spring-boot-starter-3.0.2.jar"
- "BOOT-INF/lib/HikariCP-5.0.1.jar"
- "BOOT-INF/lib/spring-jdbc-6.0.11.jar"
- "BOOT-INF/lib/spring-tx-6.0.11.jar"
- "BOOT-INF/lib/mybatis-spring-boot-autoconfigure-3.0.2.jar"
- "BOOT-INF/lib/mybatis-3.5.13.jar"
- "BOOT-INF/lib/mybatis-spring-3.0.2.jar"
- "BOOT-INF/lib/mysql-connector-j-8.0.33.jar"
- "BOOT-INF/lib/spring-boot-jarmode-layertools-3.1.3.jar"

```
整个工程的层级目录索引在"BOOT-INF/layers.idx"。
```text
- "dependencies":
  - "BOOT-INF/lib/"
- "spring-boot-loader":
  - "org/"
- "snapshot-dependencies":
- "application":
  - "BOOT-INF/classes/"
  - "BOOT-INF/classpath.idx"
  - "BOOT-INF/layers.idx"
  - "META-INF/"

```
JarLauncher源码如下。与其相对的有PropertiesLauncher和WarLauncher。
```java
public class JarLauncher extends ExecutableArchiveLauncher {

	static final EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
		if (entry.isDirectory()) {
			return entry.getName().equals("BOOT-INF/classes/");
		}
		return entry.getName().startsWith("BOOT-INF/lib/");
	};

	public JarLauncher() {
	}

	protected JarLauncher(Archive archive) {
		super(archive);
	}

	@Override
	protected boolean isPostProcessingClassPathArchives() {
		return false;
	}

	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		return NESTED_ARCHIVE_ENTRY_FILTER.matches(entry);
	}

	@Override
	protected String getArchiveEntryPathPrefix() {
		return "BOOT-INF/";
	}

	public static void main(String[] args) throws Exception {
		new JarLauncher().launch(args);
	}

}

```
# 二、@Import注解

@Import并不是SpringBoot独有的注解，是原来SpringFramework的注解。
@Import用以导入额外的Bean信息或者配置信息。<br>
看下下面的示例简单理解一下。
源码地址为[https://github.com/zouhuanli/SpringMvcDemo](https://github.com/zouhuanli/SpringMvcDemo).

1.导入普通的Bean。

```java
public class TestBeanA {
}
```

2.导入Configuration。
```java
@Configuration
public class TestConfiguration {
    @Bean(name = "testBeanB")
    public TestBeanB testBeanB() {
        return new TestBeanB();
    }
}

class TestBeanB {
}

```

3.通过ImportSelector导入额外的Bean。
```java
public class TestImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        System.out.println("TestImportSelector.selectImports:" + importingClassMetadata);
        return new String[]{TestBeanC.class.getName()};
    }

}


class TestBeanC {
}

```

4.通过ImportBeanDefinitionRegistrar导入额外的Bean。
```java
public class TestRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        System.out.println("TestRegistrar.registerBeanDefinitions");
        registry.registerBeanDefinition("testBeanD", new RootBeanDefinition(TestBeanD.class));

    }
}

class TestBeanD {
}
```

注解配置类。
```java
@Configuration
@Import({TestBeanA.class, TestConfiguration.class, TestImportSelector.class, TestRegistrar.class})
public class ImportConfig {
}
```

测试主类。
```java
 public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AnnotationContextConfig.class);
        System.out.println(applicationContext.getBean(DataSource.class));
        System.out.println(applicationContext.getBean("myFactoryBean"));
        applicationContext.getBean(UserController.class).hello(null, null, null);
        applicationContext.getBean(UserMapper.class).findUserList().forEach(System.out::println);
        System.out.println(applicationContext.getBean(DataSource.class));
        System.out.println(applicationContext.getBean(TestBeanA.class));

        System.out.println(applicationContext.getBean(TestConfiguration.class));
        System.out.println(applicationContext.getBean("testBeanB"));

       // System.out.println(applicationContext.getBean(TestImportSelector.class));
        System.out.println(applicationContext.getBean("com.homura.bean.importtest.TestBeanC"));

       // System.out.println(applicationContext.getBean(TestRegistrar.class));
        System.out.println(applicationContext.getBean("testBeanD"));

    }
```
测试结果：
```text
2023-11-06 21:01:01,508|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'com.homura.bean.importtest.TestBeanA'
com.homura.bean.importtest.TestBeanA@7b81616b
2023-11-06 21:01:01,508|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'testConfiguration'
com.homura.bean.importtest.TestConfiguration$$SpringCGLIB$$0@15d42ccb
2023-11-06 21:01:01,508|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'testBeanB'
com.homura.bean.importtest.TestBeanB@279dd959
2023-11-06 21:01:01,508|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'com.homura.bean.importtest.TestBeanC'
com.homura.bean.importtest.TestBeanC@46383a78
2023-11-06 21:01:01,508|TRACE|AbstractBeanFactory.java:256 |main|Returning cached instance of singleton bean 'testBeanD'
```
这里TestConfiguration是使用@Configurtaion注解的，本身会注册为Spring的Bean。而Selector、Registrar自身并没有注册为Spring的bean。<br>
Selector、Registrar对象创建的代码是在这里：
```java
	for (SourceClass candidate : importCandidates) {
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
```
通过ParserStrategyUtils.instantiateClass方法创建的。

# 三、@Conditional注解

我们在XXXAutoConfiguration这些类中经常遇到@ConditionalOnClass、@ConditionalOnMissingBean、@Conditional这些注解，允许在存在某些bean、某些类型的时候创建对应AutoConfiguration要创建的Bean。

以OnClassCondition为例：
```java
@Override
	public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
		ClassLoader classLoader = context.getClassLoader();
		ConditionMessage matchMessage = ConditionMessage.empty();
		List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
		if (onClasses != null) {
			List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
			if (!missing.isEmpty()) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
					.didNotFind("required class", "required classes")
					.items(Style.QUOTE, missing));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
				.found("required class", "required classes")
				.items(Style.QUOTE, filter(onClasses, ClassNameFilter.PRESENT, classLoader));
		}
		List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);
		if (onMissingClasses != null) {
			List<String> present = filter(onMissingClasses, ClassNameFilter.PRESENT, classLoader);
			if (!present.isEmpty()) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnMissingClass.class)
					.found("unwanted class", "unwanted classes")
					.items(Style.QUOTE, present));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
				.didNotFind("unwanted class", "unwanted classes")
				.items(Style.QUOTE, filter(onMissingClasses, ClassNameFilter.MISSING, classLoader));
		}
		return ConditionOutcome.match(matchMessage);
	}
```
这里判断类型是否存在，从而计算到匹配结果.

我们再看下这个注解：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

	/**
	 * All {@link Condition} classes that must {@linkplain Condition#matches match}
	 * in order for the component to be registered.
	 */
	Class<? extends Condition>[] value();
```
再找到处理这个注解的源码。
```java
/**
	 * Determine if an item should be skipped based on {@code @Conditional} annotations.
	 * @param metadata the meta data
	 * @param phase the phase of the call
	 * @return if the item should be skipped
	 */
	public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
                                //没有Conditional注解
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}
                                //若有阶段这个对象
		if (phase == null) {
			if (metadata instanceof AnnotationMetadata annotationMetadata &&
					ConfigurationClassUtils.isConfigurationCandidate(annotationMetadata)) {
				return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
			}
			return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
		}
                            //获得所有的Conditional
		List<Condition> conditions = new ArrayList<>();
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}
                             //排序
		AnnotationAwareOrderComparator.sort(conditions);
                            //计算是否匹配
		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition configurationCondition) {
				requiredPhase = configurationCondition.getConfigurationPhase();
			}
			if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
				return true;
			}
		}

		return false;
	}
```
这样就根据计算结果判断是否跳过注册某一个Bean。

# 四、参考材料

1.《Spring Boot Reference Documentation》(version 3.1.5)<br>
2.SpringBoot源码(版本3.1.3)
