---
layout: post
title: "Mybatis源码解读三:MapperProxy创建过程"
date: 2023-11-15
tags: [ mybatis ]
comments: true
author: zouhuanli
---

本文是mybatis源码阅读计划的第三篇文章,本文简单介绍一下Mybatis的初始化过程，也就是解读一下Configuration和SqlSessionFactory的启动过程。<br>

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

前面我们已经知道mybatis对MapperInterface最终会创建MapperProxy对象，用以代理接口的全部SQL方法。其创建流程如下图：

![MapperFactoryBean_getObject](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-08-mybatis_source_code_reading_1/MapperFactoryBean_getObject.png)

其创建流程的入口时MapperFactoryBean#getObject()方法，通过MapperScanner会向Spring容器注册MapperFactoryBean。MapperFactoryBean是一类FactoryBean，FactoryBean是创建Bean的工厂， 主要用以定制创建复杂对象的方法，因为Spring默认的创建对象方法的通过构造器。
FactoryBean在Spring容器注册的是FactoryBean#getObject()方法返回的对象。获取FactoryBean本身而不是getObject()方法返回的对象，需要使用&+beanName或者指定类型来获取。

本文从与Spring源码结合的角度，解读MapperFactoryBean、MapperProxy的创建过程。

## 一、从MapperScan注解谈起

我们先进入这个注解@MapperScan，用以注册mapper接口:

```java
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {
    //......
}
```

继续解读MapperScannerRegistrar这个类。
```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

    /**
     * {@inheritDoc}
     *
     * @deprecated Since 2.0.2, this method not used never.
     */
    @Override
    @Deprecated
    public void setResourceLoader(ResourceLoader resourceLoader) {
        // NOP
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AnnotationAttributes mapperScanAttrs = AnnotationAttributes
                .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
        if (mapperScanAttrs != null) {
            registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
                    generateBaseBeanName(importingClassMetadata, 0));
        }
    }

    void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
                                 BeanDefinitionRegistry registry, String beanName) {

        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
        builder.addPropertyValue("processPropertyPlaceHolders", true);

        Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
        if (!Annotation.class.equals(annotationClass)) {
            builder.addPropertyValue("annotationClass", annotationClass);
        }
    }
}
```
MapperScannerRegistrar集成自ImportBeanDefinitionRegistrar，XXXRegistrar已经见过多个了。ImportBeanDefinitionRegistrar用以向Spring容器注册额外的bean信息。
在ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars这里注册额外的Bean信息：
```java
	private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
		registrars.forEach((registrar, metadata) ->
				registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
	}
```
注意看这行" BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class)",这里向Spring容器注册了MapperScannerConfigurer。

继续来到MapperScannerConfigurer，MapperScannerConfigurer实现了BeanDefinitionRegistryPostProcessor，用以向Spring容器中注册Mapper接口,如UserMapper。
```java
@Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
                                //创建扫描器.类似于ClassPathBeanDefinitionScanner
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
      scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    if (StringUtils.hasText(defaultScope)) {
      scanner.setDefaultScope(defaultScope);
    }
    scanner.registerFilters();
                            //扫描和注册路径下的Bean信息
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```
这里的扫描方法是先由Spring(ClassPathBeanDefinitionScanner)扫描对应的路径,再由ClassPathMapperScanner处理。

```java
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      if (printWarnLogIfNotFoundMappers) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
            + "' package. Please check your configuration.");
      }
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```

最后来到processBeanDefinitions这个方法：

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    AbstractBeanDefinition definition;
    BeanDefinitionRegistry registry = getRegistry();
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (AbstractBeanDefinition) holder.getBeanDefinition();
      boolean scopedProxy = false;
      if (ScopedProxyFactoryBean.class.getName().equals(definition.getBeanClassName())) {
        definition = (AbstractBeanDefinition) Optional
            .ofNullable(((RootBeanDefinition) definition).getDecoratedDefinition())
            .map(BeanDefinitionHolder::getBeanDefinition).orElseThrow(() -> new IllegalStateException(
                "The target bean definition of scoped proxy bean not found. Root bean definition[" + holder + "]"));
        scopedProxy = true;
      }
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
          + "' mapperInterface");
                                 //注意看这行日志
      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      try {
        // for spring-native
                                //设置mapperInterface，Mapper接口
        definition.getPropertyValues().add("mapperInterface", Resources.classForName(beanClassName));
      } catch (ClassNotFoundException ignore) {
        // ignore
      }
                                //设置bean的class类为MapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBeanClass);

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      // Attribute for MockitoPostProcessor
      // https://github.com/mybatis/spring-boot-starter/issues/475
      definition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, beanClassName);
                                //下面是注入sqlSessionFactory、sqlSessionTemplate
      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory",
            new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate",
            new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }

      definition.setLazyInit(lazyInitialization);

      if (scopedProxy) {
        continue;
      }

      if (ConfigurableBeanFactory.SCOPE_SINGLETON.equals(definition.getScope()) && defaultScope != null) {
        definition.setScope(defaultScope);
      }

      if (!definition.isSingleton()) {
        BeanDefinitionHolder proxyHolder = ScopedProxyUtils.createScopedProxy(holder, registry, true);
        if (registry.containsBeanDefinition(proxyHolder.getBeanName())) {
          registry.removeBeanDefinition(proxyHolder.getBeanName());
        }
        registry.registerBeanDefinition(proxyHolder.getBeanName(), proxyHolder.getBeanDefinition());
      }

    }
  }
```
这里特别注意的是MapperInterface的bean信息中，接口类型为对应的Mapper接口如UserMapper，bean的class类型为MapperFactoryBean。MapperFactoryBean会向容器注册MapperFactoryBean#getObject()方法
返回的对象。
最终的BeanDefinition如下。

![ClassPathMapperScanner](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-08-mybatis_source_code_reading_1/ClassPathMapperScanner.png)

其注册到Spring容器的Bean的Name是userMapper、actorMapper,是Mapper接口的首字母小写。
而MapperFactoryBean本身的nam是&userMapper、&actorMapper。

## 二、创建MapperFactoryBean

我们继续来到MapperFactoryBean这个类。
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
                     //T的类型，也就是mapper接口
  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  public MapperFactoryBean() {
    // intentionally empty
  }
                         //设置mapper接口
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
                        //前置处理，注册MapperRegistry。和前一篇文章一样，最后是在knownMappers.put(type, new MapperProxyFactory<>(type));在knownMappers加入创建接口的MapperProxy的工厂。
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

  /**
   * {@inheritDoc}
   */
                        //创建对象的方法，注册到Spring容器
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
                        //创建对象的类型
  /**
   * {@inheritDoc}
   */
  @Override
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public boolean isSingleton() {
    return true;
  }

  
}
```
创建MapperFactoryBean时候通过afterPropertiesSet方法调用checkDaoConfig方法，向在knownMappers加入创建接口的MapperProxy的工厂MapperProxyFactory。
" return getSqlSession().getMapper(this.mapperInterface);"这里getSqlSession()返回的是注册的sqlSessionTemplate。

然后开始创建MapperProxy对象。


## 三、创建MapperProxy

从getMapper方法是真正创建MapperProxy对象的流程，Spring的单例bean是只会创建一次，所以Mapper接口对应的MapperProxy对象也只会创建一个，然后加入到singletonObjects中。
Mapper接口本身只有方法和SQL语句，并没有保存状态，因此创建为单例，作为整个应用周期唯一，是比较合理和节省资源的做法。

继续来到SqlSessionTemplate#getMapper方法。

```java
@Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }
```

Configuration就是之前创建的全局配置的对象,然后是Configuration#getMapper方法。

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
这里从mapperRegistry获取Mapper接口对应的MapperProxyFactory对象。

再进入MapperRegistry。
```java
public class MapperRegistry {

    private final Configuration config;
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    public MapperRegistry(Configuration config) {
        this.config = config;
    }

    @SuppressWarnings("unchecked")
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
}
```
knownMappers是之前已经解析和创建好的，这里直接获取对应接口的MapperProxyFactory。

MapperProxyFactory源码如下：

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethodInvoker> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

最终Mapper接口被创建为MapperProxy对象，并注册到Spring容器中,通过自动注入获取Mapper接口时候将会获得对应的MapperProxy对象。


## 四、参考材料

1.https://mybatis.org/mybatis-3/zh/ <br>
2.Mybatis源码(版本3.5.13) <br>
