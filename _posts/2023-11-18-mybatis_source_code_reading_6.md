---
layout: post
title: "Mybatis源码解读六:插件机制"
date: 2023-11-10
tags: [ mybatis ]
comments: true
author: zouhuanli
---

本文是mybatis源码阅读计划的第六文章,本文简单解读Mybatis插件的实现机制。

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

mybatis的插件也称为拦截器,在创建Configuration和SqlSessionFactory时，有下列代码在interceptorChain拦截器链添加拦截器：

```java
if(!isEmpty(this.plugins)){
        Stream.of(this.plugins).forEach(plugin->{
            targetConfiguration.addInterceptor(plugin);
        LOGGER.debug(()->"Registered plugin: '"+plugin+"'");
            });
        }
```
MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

1. Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
2. ParameterHandler (getParameterObject, setParameters)
3. ResultSetHandler (handleResultSets, handleOutputParameters)
4. StatementHandler (prepare, parameterize, batch, update, query)

下面从一个简单的示例按探究mybatis的实现机制。

## 一、简单使用

拦截器接口如下：

```java
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  default Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  default void setProperties(Properties properties) {
    // NOP
  }

}

```

我们手动实现一个间的的拦截器即可。

```java
/**
 * 示例拦截器，来自mybatis官网
 */
@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class ExamplePlugin implements Interceptor {
    private static final Logger LOGGER = LoggerFactory.getLogger(ExamplePlugin.class);
    private Properties properties = new Properties();

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // implement pre processing if need
        LOGGER.info("ExamplePlugin intercept");
        Object returnObject = invocation.proceed();
        // implement post processing if need
        LOGGER.info("ExamplePlugin intercept");
        return returnObject;
    }

    @Override
    public void setProperties(Properties properties) {
        this.properties = properties;
    }
}
```
这个拦截器拦截了Executor#query方法的执行，方法参数时args这个数组。

拦截器添加到Mybatis的Configuration中:

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true, mode = AdviceMode.PROXY)
@MapperScan(basePackages = "com.homura.myspringboot.dao")
public class MyBatisConfig {

    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return configuration -> {
            // 注册拦截器
            ExamplePlugin examplePlugin = new ExamplePlugin();
            configuration.addInterceptor(examplePlugin);
        };
    }
}

```

执行测试，如下。
```text
2023-11-19 20:00:51,848|INFO |   HikariDataSource.java:110 |http-nio-8888-exec-1|MyBootHikariCP - Starting...
2023-11-19 20:00:51,921|INFO |         HikariPool.java:565 |http-nio-8888-exec-1|MyBootHikariCP - Added connection com.mysql.cj.jdbc.ConnectionImpl@3b14e2ec
2023-11-19 20:00:51,922|INFO |   HikariDataSource.java:123 |http-nio-8888-exec-1|MyBootHikariCP - Start completed.
2023-11-19 20:00:51,931|INFO |      ExamplePlugin.java:33  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,943|INFO |      ExamplePlugin.java:36  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,943|INFO |      ExamplePlugin.java:33  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,944|INFO |      ExamplePlugin.java:36  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,944|INFO |      ExamplePlugin.java:33  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,944|INFO |      ExamplePlugin.java:36  |http-nio-8888-exec-1|ExamplePlugin intercept
2023-11-19 20:00:51,970|INFO |           MyFilter.java:44  |http-nio-8888-exec-1|)MyFilter.afterDo(
2023-11-19 20:02:29,948|INFO |         MyListener.java:33  |SpringApplicationShutdownHook|MyListener.contextDestroyed()
```

下面解读一下拦截器的实现源码。

## 二、源码解读

### 2.1 注册拦截器到拦截器链

在创建SqlSessionFactory和Configuration时候，会解析XML或者注解注册的拦截器列表
```java

    if (!isEmpty(this.plugins)) {
      Stream.of(this.plugins).forEach(plugin -> {
        targetConfiguration.addInterceptor(plugin);
        LOGGER.debug(() -> "Registered plugin: '" + plugin + "'");
      });
    }
```

加入到Configuration内部的拦截器链里面：

```java
  protected final InterceptorChain interceptorChain = new InterceptorChain();

```

拦截器链就是一个简单的ArrayList。

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```

注册拦截器到拦截器链之后，下面看下拦截器在何处起作用。

### 2.2 拦截器作用位置

观察pluginAll的引用代码，分别Executor、ParameterHandler、ResultSetHandler、StatementHandler，是共有四个：

![pluginAll](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-19-mybatis_source_code_reading_6/pluginAll.png)

这里按示例的Executor为例,来到Configuration#newExecutor方法。

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    return (Executor) interceptorChain.pluginAll(executor);
  }

```

这里interceptorChain#pluginAll如下，这里依次调用拦截器来创建代理对象。

```java
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
```

最后来到Plugin#wrap方法。

```java
public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(type.getClassLoader(), interfaces, new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```
这里Plugin实现了InvocationHandler，使用JDK动态代理来创建包含拦截器对象、目标对象的代理对象。<br>
注意这里是在for循环里面执行的，因此每一个拦截器/插件都会创建一次代理对象。<br>
所以最后拦截器/插件是通过JDK动态代理创建原始对象的代理对象来起作用的。


## 三、PageHelper简单解读

PageHelper一款用以分页的mybatis插件，以starter的方法引入：

```xml
  <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>
```

PageHelperAutoConfiguration在这里向拦截器链添加了PageInterceptor分页插件。

```java
 @Override
public void afterPropertiesSet() throws Exception {
        PageInterceptor interceptor = new PageInterceptor();
        interceptor.setProperties(this.properties);
        for (SqlSessionFactory sqlSessionFactory : sqlSessionFactoryList) {
            org.apache.ibatis.session.Configuration configuration = sqlSessionFactory.getConfiguration();
        if (!containsInterceptor(configuration, interceptor)) {
                configuration.addInterceptor(interceptor);
            }
            }
        }
```

其配置类为PageHelperProperties。

简单测试如下：
```java

    /**
     * 查找全部
     *
     * @return
     */
    @Transactional(rollbackFor = Exception.class)
    public List<User> listAll() {
        //仅供测试
        PageHelper.startPage(1, 5);
        List<User> list = userMapper.findAll();
        userMapper.findAll();
        userMapper.findAll();
        return list;
    }
```
测试返回结果如下：
````json
[
  {
    "uid": 3,
    "uname": "bb",
    "password": "123",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  },
  {
    "uid": 7,
    "uname": "L",
    "password": "123",
    "gender": "男",
    "phone": "12345678900",
    "email": "123@qq.com",
    "address": "我家",
    "age": null
  },
  {
    "uid": 8,
    "uname": "123",
    "password": "213",
    "gender": "男",
    "phone": "321",
    "email": "213",
    "address": "3",
    "age": null
  },
  {
    "uid": 11,
    "uname": "",
    "password": "",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  },
  {
    "uid": 12,
    "uname": "111",
    "password": "222",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  }
]
````
数据库包含11条数据，这里测试只查询第一页5条数据。<br>其原理就是拦截查询方法，增加分页的SQL参数。
在ExecutorUtil#pageQuery方法：
```java
 /**
     * 分页查询
     *
     * @param dialect
     * @param executor
     * @param ms
     * @param parameter
     * @param rowBounds
     * @param resultHandler
     * @param boundSql
     * @param cacheKey
     * @param <E>
     * @return
     * @throws SQLException
     */
    public static <E> List<E> pageQuery(Dialect dialect, Executor executor, MappedStatement ms, Object parameter,
                                        RowBounds rowBounds, ResultHandler resultHandler,
                                        BoundSql boundSql, CacheKey cacheKey) throws SQLException {
        //判断是否需要进行分页查询
        if (dialect.beforePage(ms, parameter, rowBounds)) {
            //生成分页的缓存 key
            CacheKey pageKey = cacheKey;
            //处理参数对象
            parameter = dialect.processParameterObject(ms, parameter, boundSql, pageKey);
            //调用方言获取分页 sql
            String pageSql = dialect.getPageSql(ms, boundSql, parameter, rowBounds, pageKey);
            BoundSql pageBoundSql = new BoundSql(ms.getConfiguration(), pageSql, boundSql.getParameterMappings(), parameter);

            Map<String, Object> additionalParameters = getAdditionalParameter(boundSql);
            //设置动态参数
            for (String key : additionalParameters.keySet()) {
                pageBoundSql.setAdditionalParameter(key, additionalParameters.get(key));
            }
            //对 boundSql 的拦截处理
            if (dialect instanceof BoundSqlInterceptor.Chain) {
                pageBoundSql = ((BoundSqlInterceptor.Chain) dialect).doBoundSql(BoundSqlInterceptor.Type.PAGE_SQL, pageBoundSql, pageKey);
            }
            //执行分页查询
            return executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, pageKey, pageBoundSql);
        } else {
            //不执行分页的情况下，也不执行内存分页
            return executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, cacheKey, boundSql);
        }
    }
```
这里原始的boundSql是"select * from  user"，

添加分页参数后的pageBoundSql如下：

![pageBoundSql](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-19-mybatis_source_code_reading_6/pageBoundSql.png)


总结的说，就是mybatis的插件基于拦截器创建代理对象来实现，可以拦截Executor、ParameterHandler、ResultSetHandler、StatementHandler的方法，来实现开发者定制化的功能。

## 四、参考材料

1.https://mybatis.org/mybatis-3/zh/ <br>
2.Mybatis源码(版本3.5.13) <br>

