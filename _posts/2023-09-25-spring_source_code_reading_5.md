---
layout: post
title: "Spring源码阅读五:AnnotationConfigApplicationContext初始化流程"
date: 2023-09-25
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第五篇文章，本文主要介绍Spring IoC容器中AnnotationConfigApplicationContext的初始化流程。上一篇已经介绍了基于XML
的ClassPathXmlApplicationContext的初始化流程，本篇的基于注解的ApplicationContext初始化流程整体和上一篇文章一致，不过也有一些不同点，本文将从源码
探讨AnnotationConfigApplicationContext的初始化流程。
本文源码地址为:[https://github.com/zouhuanli/SpringCircularReferenceDemo.git](https://github.com/zouhuanli/SpringCircularReferenceDemo.git).<br>

# 一、入口

依旧从一个最简单的例子入手：

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

和上文一样，我们从AnnotationConfigApplicationContext的构造方法作为入口开始阅读代码：

```java
 public AnnotationConfigApplicationContext(Class<?>...componentClasses){
        //构造器
        this();
        //注册配置类，这个成员类型特指@Configuration注解的类型
        register(componentClasses);
        //刷新IoC容器，并初始化和启动ApplicationContext，和前一篇文章一样。
        refresh();
        }
```

我们看下构造器方法：

```java

/**
 * Create a new AnnotationConfigApplicationContext that needs to be populated
 * through {@link #register} calls and then manually {@linkplain #refresh refreshed}.
 */
public AnnotationConfigApplicationContext(){
        StartupStep createAnnotatedBeanDefReader=this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
        this.reader=new AnnotatedBeanDefinitionReader(this);
        createAnnotatedBeanDefReader.end();
        this.scanner=new ClassPathBeanDefinitionScanner(this);
        }
```

这里设置了Bean配置的读取器和扫描器,扫描器主要是读取扫描路径下面注解配置的Bean配置。
我们从register一路DEBUG最后进入BeanFactory的registerBeanDefinition方法，找到注册AppConfig这个配置类的配置，如下：

![AnnotationAppConfig](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-25-spring_source_code_reading_5/AnnotationAppConfig.png)

我们看到不仅仅注册了AppConfig这个配置，还注册了四个内部的配置类。<br>
下面我们开始阅读最主要的refresh方法。

# 二、refresh方法

和上一篇文章一样：

```java
public void refresh()throws BeansException,IllegalStateException{
        StartupStep contextRefresh=this.applicationStartup.start("spring.context.refresh");
        //准备刷新，设置closed、active，初始化了一些应用的监听器的集合
        // Prepare this context for refreshing.
        prepareRefresh();

        //销毁旧的beanFactory，创建新的beanFactory，如果是refresh的ApplicationContext，重新解析、加载、注册BeanDefinition。
        // Tell the subclass to refresh the internal bean factory.
        //GenericApplicationContext的refreshBeanFactory方法啥都没干，就设置了serializationId。我们继续往下看
        ConfigurableListableBeanFactory beanFactory=obtainFreshBeanFactory();

        //准备beanFactory，配置了类加载器、后置处理器、忽略注入的接口等等
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        //beanFactory的后置处理，空方法，框架预留的拓展点
        // Allows post-processing of the bean factory in context subclasses.
        postProcessBeanFactory(beanFactory);

        StartupStep beanPostProcess=this.applicationStartup.start("spring.context.beans.post-process");
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

我们跟踪debug流程进入下面方法：

```java
    /**
 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before singleton instantiation.
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory){
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory,getBeanFactoryPostProcessors());

        // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
        // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
        if(!NativeDetector.inNativeImage()&&beanFactory.getTempClassLoader()==null&&
        beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)){
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
        }
```

这里调用了BeanFactory自身的PostProcessor后置处理器，而不是Bean的PostProcessor后置处理器，这是两个层级的PostProcessor。<br>
我们继续跟踪，进入invokeBeanDefinitionRegistryPostProcessors方法:

```java
    /**
 * Invoke the given BeanDefinitionRegistryPostProcessor beans.
 */
private static void invokeBeanDefinitionRegistryPostProcessors(
        Collection<?extends BeanDefinitionRegistryPostProcessor> postProcessors,BeanDefinitionRegistry registry,ApplicationStartup applicationStartup){

        for(BeanDefinitionRegistryPostProcessor postProcessor:postProcessors){
        StartupStep postProcessBeanDefRegistry=applicationStartup.start("spring.context.beandef-registry.post-process")
        .tag("postProcessor",postProcessor::toString);
        postProcessor.postProcessBeanDefinitionRegistry(registry);
        postProcessBeanDefRegistry.end();
        }
        }
```

继续向前阅读，进入processConfigBeanDefinitions方法：
我们看到作为配置类的Bean信息有这些个：

![configCandidates](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-25-spring_source_code_reading_5/configCandidates.png)

我们继续调试后面的代码。

```java

// Parse each @Configuration class
		ConfigurationClassParser parser=new ConfigurationClassParser(
                this.metadataReaderFactory,this.problemReporter,this.environment,
                this.resourceLoader,this.componentScanBeanNameGenerator,registry);
                //这里先加入AppConfig类自身。
                Set<BeanDefinitionHolder> candidates=new LinkedHashSet<>(configCandidates);
```

这里configCandidates是@Configuration的类，先加入到candidates候选Bean配置集合，@Configuration配置的类自身也是一个Bean。<br>
然后我们进入ConfigurationClassParser的parse(Set<BeanDefinitionHolder> configCandidates)方法，这个方法非常重要，它从一个配置类读取
所有由其配置的Bean配置。
我们进入parse方法后继续跟踪：

```java
protected void processConfigurationClass(ConfigurationClass configClass,Predicate<String> filter)throws IOException{
        if(this.conditionEvaluator.shouldSkip(configClass.getMetadata(),ConfigurationPhase.PARSE_CONFIGURATION)){
        return;
        }

        ConfigurationClass existingClass=this.configurationClasses.get(configClass);
        if(existingClass!=null){
        if(configClass.isImported()){
        if(existingClass.isImported()){
        existingClass.mergeImportedBy(configClass);
        }
        // Otherwise ignore new imported config class; existing non-imported class overrides it.
        return;
        }
        else{
        // Explicit bean definition found, probably replacing an import.
        // Let's remove the old one and go with the new one.
        this.configurationClasses.remove(configClass);
        this.knownSuperclasses.values().removeIf(configClass::equals);
        }
        }

        // Recursively process the configuration class and its superclass hierarchy.
        SourceClass sourceClass=asSourceClass(configClass,filter);
        do{
        //进入这里
        sourceClass=doProcessConfigurationClass(configClass,sourceClass,filter);
        }
        while(sourceClass!=null);

        this.configurationClasses.put(configClass,configClass);
        }
```

我们接下来看到<strong>ComponentScanAnnotationParser.doProcessConfigurationClass</strong>方法：

```java

protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass,SourceClass sourceClass,Predicate<String> filter)
        throws IOException{

        if(configClass.getMetadata().isAnnotated(Component.class.getName())){
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass,sourceClass,filter);
        }

        // Process any @PropertySource annotations
        for(AnnotationAttributes propertySource:AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(),PropertySources.class,
        org.springframework.context.annotation.PropertySource.class)){
        if(this.propertySourceRegistry!=null){
        this.propertySourceRegistry.processPropertySource(propertySource);
        }
        else{
        logger.info("Ignoring @PropertySource annotation on ["+sourceClass.getMetadata().getClassName()+
        "]. Reason: Environment must implement ConfigurableEnvironment");
        }
        }

        // Process any @ComponentScan annotations,处理成员扫描，从扫描路径读取全部Bean配置
        Set<AnnotationAttributes> componentScans=AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(),ComponentScans.class,ComponentScan.class);
        if(!componentScans.isEmpty()&&
        !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(),ConfigurationPhase.REGISTER_BEAN)){
        for(AnnotationAttributes componentScan:componentScans){
        // The config class is annotated with @ComponentScan -> perform the scan immediately
        //扫描成员的Bean配置的重点的代码
        Set<BeanDefinitionHolder> scannedBeanDefinitions=
        this.componentScanParser.parse(componentScan,sourceClass.getMetadata().getClassName());
        // Check the set of scanned definitions for any further config classes and parse recursively if needed
        for(BeanDefinitionHolder holder:scannedBeanDefinitions){
        BeanDefinition bdCand=holder.getBeanDefinition().getOriginatingBeanDefinition();
        if(bdCand==null){
        bdCand=holder.getBeanDefinition();
        }
        if(ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand,this.metadataReaderFactory)){
        parse(bdCand.getBeanClassName(),holder.getBeanName());
        }
        }
        }
        }

        // Process any @Import annotations ，处理@import
        processImports(configClass,sourceClass,getImports(sourceClass),filter,true);

        // Process any @ImportResource annotations
        AnnotationAttributes importResource=
        AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(),ImportResource.class);
        if(importResource!=null){
        String[]resources=importResource.getStringArray("locations");
        Class<?extends BeanDefinitionReader> readerClass=importResource.getClass("reader");
        for(String resource:resources){
        String resolvedResource=this.environment.resolveRequiredPlaceholders(resource);
        configClass.addImportedResource(resolvedResource,readerClass);
        }
        }

        // Process individual @Bean methods，处理@Bean单独创建的Bean的配置
        Set<MethodMetadata> beanMethods=retrieveBeanMethodMetadata(sourceClass);
        for(MethodMetadata methodMetadata:beanMethods){
        configClass.addBeanMethod(new BeanMethod(methodMetadata,configClass));
        }

        // Process default methods on interfaces
        processInterfaces(configClass,sourceClass);

        // Process superclass, if any
        if(sourceClass.getMetadata().hasSuperClass()){
        String superclass=sourceClass.getMetadata().getSuperClassName();
        if(superclass!=null&&!superclass.startsWith("java")&&
        !this.knownSuperclasses.containsKey(superclass)){
        this.knownSuperclasses.put(superclass,configClass);
        // Superclass found, return its annotation metadata and recurse
        return sourceClass.getSuperClass();
        }
        }

        // No superclass -> processing is complete
        return null;
        }
```

我们继续跟踪这行代码"Set<BeanDefinitionHolder> scannedBeanDefinitions =
this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());",
来到<strong>"ClassPathBeanDefinitionScanner.doScan(String... basePackages)"</strong>方法，这个方法描述了如何从扫描路径读取所有的
Bean配置。doScan读取了扫描路径所有的Bean配置，解析为BeanDefinition，并最终注册到DefaultListableBeanFactory的beanDefinitionMap(和上一篇一样)

```java
   protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
            //从basePackage解析所有的BeanDefinition
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
        candidate.setScope(scopeMetadata.getScopeName());
        String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
        if (candidate instanceof AbstractBeanDefinition abstractBeanDefinition) {
        postProcessBeanDefinition(abstractBeanDefinition, beanName);
        }
        if (candidate instanceof AnnotatedBeanDefinition annotatedBeanDefinition) {
        AnnotationConfigUtils.processCommonDefinitionAnnotations(annotatedBeanDefinition);
        }
        //判断是否符合
        if (checkCandidate(beanName, candidate)) {
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
        definitionHolder =
        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
        beanDefinitions.add(definitionHolder);
        //注册到DefaultListableBeanFactory的beanDefinitionMap
        registerBeanDefinition(definitionHolder, this.registry);
        }
        }
        }
        return beanDefinitions;
        }         
```
我们继续进入代码，来阅读scanCandidateComponents方法(有删减)：

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage){
        Set<BeanDefinition> candidates=new LinkedHashSet<>();

        String packageSearchPath=ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX+
        resolveBasePackage(basePackage)+'/'+this.resourcePattern;
        //解析所有的资源，也就是扫描路径下面的类
        Resource[]resources=getResourcePatternResolver().getResources(packageSearchPath);
        //对每个类解析为ScannedGenericBeanDefinition
        for(Resource resource:resources){
        String filename=resource.getFilename();

        MetadataReader metadataReader=getMetadataReaderFactory().getMetadataReader(resource);
        if(isCandidateComponent(metadataReader)){
        //
        ScannedGenericBeanDefinition sbd=new ScannedGenericBeanDefinition(metadataReader);
        sbd.setSource(resource);
        if(isCandidateComponent(sbd)){
        //BeanDefinition添加到候选集合
        candidates.add(sbd);
                }
            }

         }

        return candidates;
        }
```

看下resources的value:

![resources](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-25-spring_source_code_reading_5/resources.png)

然后我们看下这里解析处理的BeanDefinition的值；

![ScannedGenericBeanDefinition](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-09-25-spring_source_code_reading_5/ScannedGenericBeanDefinition.png)

好了，到这里现在basePackage路径下面的所有注册配置的Bean配置都已经解析为BeanDefinition了并返回到外部。<br>
后面就是registerBeanDefinition方法了注册到beanDefinitionMap，本篇不再赘述。

# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)  <br>
