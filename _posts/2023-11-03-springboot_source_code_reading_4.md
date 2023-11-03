---
layout: post
title: "SpringBoot源码阅读四:事务功能和Mybatis集成"
date: 2023-11-02
tags: [ springboot ]
comments: true
author: zouhuanli
---

本文是SpringBoot源码阅读计划的第三篇文章，本文简单解读一下SpringBoot开启注解事务和Mybatis集成的源代码。<br>
本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>
本文应该算不上是源码解读，会更侧重具体用法。创建某一类功能的组件以及自动配置的源码在上一篇文章有解读，开启注解事务、创建事务管理器、以及集成Mybatis的流程，整体上和注解配置一样,
只不过SpringBoot自动创建所需要的Bean，以及自动读取和存储配置。


# 一、注解配置

我们使用注册开启事务和配置Mybatis可以如下。
```java
/**
 * 注解方式mybatis配置类
 */
@Configuration
                                                    //开启事务注解事务，XML配置是 <tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" mode="proxy"/>这样
@EnableTransactionManagement(proxyTargetClass = true, mode = AdviceMode.PROXY)
                                                    // 设置自动代理的proxy-target-class， <aop:aspectj-autoproxy proxy-target-class="true"/>
@EnableAspectJAutoProxy(proxyTargetClass = true)
                                                    //配置MapperScannerConfigurer，自动扫描Mapper接口
@MapperScan(basePackages = "com.homura.dao", sqlSessionTemplateRef = "sqlSessionTemplate", sqlSessionFactoryRef = "sqlSessionFactory")
public class MyBatisConfig {
                                                    //创建事务管理器
    @Bean(name = "transactionManager")
    public DataSourceTransactionManager transactionManager(
            @Autowired @Qualifier("dataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
                                                //创建sqlSessionFactory
    @Bean(name = "sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(@Autowired @Qualifier("dataSource") DataSource dataSource) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);

        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
                                                //mybatis-config配置文件
        sessionFactory.setConfigLocation(resolver.getResource("classpath:mybatis-config.xml"));
        sessionFactory.setMapperLocations(resolver.getResources("classpath:/mappers/*.xml"));//配置Mapper映射文件的路径
        return sessionFactory.getObject();
    }
                                            //创建sqlSessionTemplate
    @Bean(name = "sqlSessionTemplate")
    public SqlSessionTemplate sqlSessionTemplate(@Autowired @Qualifier("sqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory);
        return sqlSessionTemplate;
    }

}
```
这样就开启事务和配置好Mybatis了，SpringBoot也是做了同样的事务，但是考虑情况更加全面，而且更具有拓展性。

# 二、事务配置
 
在上一篇文章笔者已经解读了AutoConfiguration类的注册和创建，已经对应的PropertiesBean的注册和创建、读取和存储配置文件的配置。这里将会简化一下解读本文涉及的AutoConfiguration类。

## 2.1 DataSourceTransactionManagerAutoConfiguration自动配置类

在引入starter-jdbc这个依赖之后，会引入这个自动配置类。其源码如下。
```java
                                    //自动配置类，在TransactionAutoConfiguration之前
@AutoConfiguration(before = TransactionAutoConfiguration.class)
@ConditionalOnClass({ JdbcTemplate.class, TransactionManager.class })
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
                                    //获得对应PropertiesBean
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceTransactionManagerAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnSingleCandidate(DataSource.class)
	static class JdbcTransactionManagerConfiguration {
                                     //这里创建了事务管理器
		@Bean
		@ConditionalOnMissingBean(TransactionManager.class)
		DataSourceTransactionManager transactionManager(Environment environment, DataSource dataSource,
				ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
			DataSourceTransactionManager transactionManager = createTransactionManager(environment, dataSource);
			transactionManagerCustomizers.ifAvailable((customizers) -> customizers.customize(transactionManager));
			return transactionManager;
		}

		private DataSourceTransactionManager createTransactionManager(Environment environment, DataSource dataSource) {
			return environment.getProperty("spring.dao.exceptiontranslation.enabled", Boolean.class, Boolean.TRUE)
					? new JdbcTransactionManager(dataSource) : new DataSourceTransactionManager(dataSource);
		}

	}

}
```
可以看到这个配置类主要功能就是创建事务管理器。我们继续来到TransactionAutoConfiguration这个类。

## 2.2  TransactionAutoConfiguration

```java
@AutoConfiguration
@ConditionalOnClass(PlatformTransactionManager.class)
@EnableConfigurationProperties(TransactionProperties.class)
public class TransactionAutoConfiguration {
                                             //定制化器
	@Bean
	@ConditionalOnMissingBean
	public TransactionManagerCustomizers platformTransactionManagerCustomizers(
			ObjectProvider<PlatformTransactionManagerCustomizer<?>> customizers) {
		return new TransactionManagerCustomizers(customizers.orderedStream().toList());
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnSingleCandidate(ReactiveTransactionManager.class)
	public TransactionalOperator transactionalOperator(ReactiveTransactionManager transactionManager) {
		return TransactionalOperator.create(transactionManager);
	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnSingleCandidate(PlatformTransactionManager.class)
	public static class TransactionTemplateConfiguration {

		@Bean
		@ConditionalOnMissingBean(TransactionOperations.class)
		public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
			return new TransactionTemplate(transactionManager);
		}

	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnBean(TransactionManager.class)
	@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
	public static class EnableTransactionManagementConfiguration {
                                            //设置创建代理是Jdk还是Cglib，设置proxy-target-class
		@Configuration(proxyBeanMethods = false)
		@EnableTransactionManagement(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
		public static class JdkDynamicAutoProxyConfiguration {

		}

		@Configuration(proxyBeanMethods = false)
                                        //注意这行，@EnableTransactionManagement注解和上面手动开启这个注解是一样的
		@EnableTransactionManagement(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
				matchIfMissing = true)
		public static class CglibAutoProxyConfiguration {

		}

	}

}

```
这里创建了TransactionManagerCustomizers、TransactionTemplate等组件，最重要的就是开启了事务注解@EnableTransactionManagement。

# 三、Mybatis集成

## 3.1 简单使用

SpringBoot集成Mybatis是非常简单的。<br>

1.首先引入mybatis的starter

```java
  <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${mybatis-spring-boot-starter.version}</version>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>${mysql-connector.version}</version>
        </dependency>
```

2.创建mapper的xml配置文件

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.homura.myspringboot.dao.UserMapper">
    <resultMap id="BaseResultMap" type="com.homura.myspringboot.entity.dvo.User">
        <id column="uid" property="uid" jdbcType="INTEGER"/>
        <result column="uname" property="uname" jdbcType="VARCHAR"/>
        <result column="password" property="password" jdbcType="VARCHAR"/>
        <result column="gender" property="gender" jdbcType="VARCHAR"/>
        <result column="phone" property="phone" jdbcType="VARCHAR"/>
        <result column="email" property="email" jdbcType="VARCHAR"/>
        <result column="address" property="address" jdbcType="VARCHAR"/>
    </resultMap>
    <select id="getUserList" resultMap="BaseResultMap">
        select *
        from user
    </select>

</mapper>

```
这里是混合注解和xml方式使用mybatis，两者各有千秋，这里不做过多比较。

3.创建Mapper(DAO)接口

```java

@Repository
public interface UserMapper {
    /**
     * 查找全部用户
     * <p>
     * 不要使用 select  *
     *
     * @return
     */
    @Select("select * from user")
    @ResultMap("BaseResultMap")
    List<User> findAll();

    /**
     * 添加用户
     *
     * @param user 用户
     * @return
     */
    @Insert(" insert  into user (uid,uname,password,gender,phone,email,address) " +
            " values " +
            " (#{user.uid},#{user.uname},#{user.password},#{user.gender},#{user.phone},#{user.email},#{user.address})")
    @ResultType(int.class)
    int save(@Param("user") User user);


    /**
     * 根据id查询用户
     * 不要使用 select  *
     *
     * @param uid 用户id
     * @return
     */
                                            //混合使用注解和xml
    @Select("select * from user where uid = #{uid}")
    @Results(id = "BaseResultMapByAnnotation", value =
            {@Result(id = true, property = "uid", column = "uid", jdbcType = JdbcType.INTEGER)
                    , @Result(property = "uname", column = "uname", jdbcType = JdbcType.VARCHAR)
                    , @Result(property = "password", column = "password", jdbcType = JdbcType.VARCHAR)
                    , @Result(property = "gender", column = "gender", jdbcType = JdbcType.VARCHAR)
                    , @Result(property = "phone", column = "phone", jdbcType = JdbcType.VARCHAR)
                    , @Result(property = "email", column = "email", jdbcType = JdbcType.VARCHAR)
                    , @Result(property = "address", column = "address", jdbcType = JdbcType.VARCHAR)})
    User selectByPrimaryKey(@Param("uid") Integer uid);
}
```

这样就可以在Service层使用DAO组件了,不要在Controller/API层使用DAO，要遵循分层架构的原则。
```java

@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;

    /**
     * 添加
     *
     * @param user
     */
    public void add(User user) {
        userMapper.save(user);
    }

    /**
     * 查找全部
     *
     * @return
     */
    public List<User> listAll() {
        return userMapper.findAll();
    }


    /**
     * 按用户ID查询
     *
     * @param uid
     * @return
     */
    public User getByUid(Integer uid) {
        return userMapper.selectByPrimaryKey(uid);
    }
}
```
可以看到这里我们也不需要mybatis-config文件了。

## 3.2 Mybatis自动配置类MybatisAutoConfiguration

MybatisAutoConfiguration的运行原理和DataSourceAutoConfiguration一样的,注册、解析对应的PropertiesBean，获得配置文件的配置，然后创建所需要的组件。<br>
其源码如下
```java
@org.springframework.context.annotation.Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
                                    //mybatis的配置对应的PropertiesBean
@EnableConfigurationProperties(MybatisProperties.class)
                                    //在DataSourceAutoConfiguration、MybatisLanguageDriverAutoConfiguration之后起作用
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {

  private static final Logger logger = LoggerFactory.getLogger(MybatisAutoConfiguration.class);

  private final MybatisProperties properties;
                                     //拦截器列表
  private final Interceptor[] interceptors;

  private final TypeHandler[] typeHandlers;

  private final LanguageDriver[] languageDrivers;

  private final ResourceLoader resourceLoader;

  private final DatabaseIdProvider databaseIdProvider;

  private final List<ConfigurationCustomizer> configurationCustomizers;

  private final List<SqlSessionFactoryBeanCustomizer> sqlSessionFactoryBeanCustomizers;

  public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider,
      ObjectProvider<TypeHandler[]> typeHandlersProvider, ObjectProvider<LanguageDriver[]> languageDriversProvider,
      ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider,
      ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider,
      ObjectProvider<List<SqlSessionFactoryBeanCustomizer>> sqlSessionFactoryBeanCustomizers) {
    this.properties = properties;
    this.interceptors = interceptorsProvider.getIfAvailable();
    this.typeHandlers = typeHandlersProvider.getIfAvailable();
    this.languageDrivers = languageDriversProvider.getIfAvailable();
    this.resourceLoader = resourceLoader;
    this.databaseIdProvider = databaseIdProvider.getIfAvailable();
    this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
    this.sqlSessionFactoryBeanCustomizers = sqlSessionFactoryBeanCustomizers.getIfAvailable();
  }

  @Override
  public void afterPropertiesSet() {
    checkConfigFileExists();
  }

  private void checkConfigFileExists() {
    if (this.properties.isCheckConfigLocation() && StringUtils.hasText(this.properties.getConfigLocation())) {
      Resource resource = this.resourceLoader.getResource(this.properties.getConfigLocation());
      Assert.state(resource.exists(),
          "Cannot find config location: " + resource + " (please add config file or check your Mybatis configuration)");
    }
  }
                                        //创建SqlSessionFactory
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    //......
    applySqlSessionFactoryBeanCustomizers(factory);
    return factory.getObject();
  }

  

  private void applySqlSessionFactoryBeanCustomizers(SqlSessionFactoryBean factory) {
    if (!CollectionUtils.isEmpty(this.sqlSessionFactoryBeanCustomizers)) {
      for (SqlSessionFactoryBeanCustomizer customizer : this.sqlSessionFactoryBeanCustomizers) {
        customizer.customize(factory);
      }
    }
  }
                                //创建SqlSessionTemplate
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
                    //创建默认的MapperScanner，默认只会扫描SpringBoot的包路径，推荐使用MapperScan注解指定Mapper的扫描路径
  /**
   * This will just scan the same base package as Spring Boot does. If you want more power, you can explicitly use
   * {@link org.mybatis.spring.annotation.MapperScan} but this will get typed mappers working correctly, out-of-the-box,
   * similar to using Spring Data JPA repositories.
   */
  public static class AutoConfiguredMapperScannerRegistrar
      implements BeanFactoryAware, EnvironmentAware, ImportBeanDefinitionRegistrar {

    private BeanFactory beanFactory;
    private Environment environment;

        //......

  }

  /**
   * If mapper registering configuration or mapper scanning configuration not present, this configuration allow to scan
   * mappers based on the same component-scanning path as Spring Boot itself.
   */
  @org.springframework.context.annotation.Configuration(proxyBeanMethods = false)
  @Import(AutoConfiguredMapperScannerRegistrar.class)
  @ConditionalOnMissingBean({ MapperFactoryBean.class, MapperScannerConfigurer.class })
  public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
      logger.debug(
          "Not found configuration for registering mapper bean using @MapperScan, MapperFactoryBean and MapperScannerConfigurer.");
    }

  }

}
```
可以看到这里和注解方式是基本一致的。<br>
再看下MybatisProperties这个mybatis的配置对象，设置了配置项的前缀是mybatis,如'mybatis:mapper-locations: classpath:mappers/*.xml'设置了Mapper的xml文件路径。

```java
@ConfigurationProperties(prefix = MybatisProperties.MYBATIS_PREFIX)
public class MybatisProperties {

  public static final String MYBATIS_PREFIX = "mybatis";

  private static final ResourcePatternResolver resourceResolver = new PathMatchingResourcePatternResolver();

  /**
   * Location of MyBatis xml config file.
   */
  private String configLocation;

  /**
   * Locations of MyBatis mapper files.
   */
  private String[] mapperLocations;

  /**
   * Packages to search type aliases. (Package delimiters are ",; \t\n")
   */
  private String typeAliasesPackage;

  /**
   * The super class for filtering type alias. If this not specifies, the MyBatis deal as type alias all classes that
   * searched from typeAliasesPackage.
   */
  private Class<?> typeAliasesSuperType;

  /**
   * Packages to search for type handlers. (Package delimiters are ",; \t\n")
   */
  private String typeHandlersPackage;

  /**
   * Indicates whether perform presence check of the MyBatis xml config file.
   */
  private boolean checkConfigLocation = false;

  /**
   * Execution mode for {@link org.mybatis.spring.SqlSessionTemplate}.
   */
  private ExecutorType executorType;

  /**
   * The default scripting language driver class. (Available when use together with mybatis-spring 2.0.2+)
   */
  private Class<? extends LanguageDriver> defaultScriptingLanguageDriver;

  /**
   * Externalized properties for MyBatis configuration.
   */
  private Properties configurationProperties;

  /**
   * A Configuration object for customize default settings. If {@link #configLocation} is specified, this property is
   * not used.
   */
  private CoreConfiguration configuration;
    //......
```

到这里开启事务功能和mybatis集成就完成了。

# 四、参考材料

1.《Spring Boot Reference Documentation》(version 3.1.5)<br>
2.SpringBoot源码(版本3.1.3)
