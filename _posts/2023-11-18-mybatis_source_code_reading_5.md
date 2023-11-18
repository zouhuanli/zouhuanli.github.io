---
layout: post
title: "Mybatis源码解读五:一级缓存和二级缓存"
date: 2023-11-18
tags: [ mybatis ]
comments: true
author: zouhuanli
---

本文是mybatis源码阅读计划的第四篇文章,本文简单解读mybatis的一级缓存、二级缓存的实现源码。<br>
Cache的思想在计算机世界随处可见，利用时间和空间的局限性，极大的提高了查询速度，也带领了一致性的问题。

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

## 一 、一级缓存

上一篇文章中解读CachingExecutor#query有提及到这个缓存。

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler)
      throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

一级缓存的默认作用域是Session级别的：

```java
public enum LocalCacheScope {
  SESSION, STATEMENT
}

```

```java

    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));

```

下面按照Session级别的作用域简单解读mybatis的一级缓存。

### 1.1 简单使用

一级缓存是默认开启的，没有相关配置的参数。因为默认一个SQL方法就会创建一个SqlSession，因此调试一级缓存可以使用@@Transactional开启事务处理。

先关闭二级缓存，配置如下：

```yaml
mybatis:
  mapper-locations: classpath:mappers/*.xml
  configuration:
    cache-enabled: false
    use-generated-keys: false
    default-executor-type: simple
    log-prefix: MYBATIS
    local-cache-scope: session
```

测试方法如下,仅供测试。

```java

/**
 * 查找全部
 *
 * @return
 */
@Transactional(rollbackFor = Exception.class)
public List<User> listAll() {
        List<User> list = userMapper.findAll();
        userMapper.findAll();
        userMapper.findAll();
        return list;
        }
```

启动应用开始测试,输入：http://localhost:8888/user/listAll
<br>返回结果：
```json
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
  },
  {
    "uid": 13,
    "uname": "333",
    "password": "333",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  },
  {
    "uid": 14,
    "uname": "666",
    "password": "666",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  },
  {
    "uid": 15,
    "uname": "777",
    "password": "777",
    "gender": null,
    "phone": null,
    "email": null,
    "address": null,
    "age": null
  },
  {
    "uid": 16,
    "uname": "16",
    "password": "16",
    "gender": "女",
    "phone": "1340",
    "email": "11",
    "address": "16道路",
    "age": null
  },
  {
    "uid": 17,
    "uname": "17",
    "password": "17",
    "gender": "男",
    "phone": "111",
    "email": "11",
    "address": "111",
    "age": null
  },
  {
    "uid": 18,
    "uname": "19",
    "password": "20",
    "gender": "222",
    "phone": "222",
    "email": "222",
    "address": "222",
    "age": null
  }
]
```

观察输出日志：
```text
2023-11-18 20:07:36,842|DEBUG|     BaseJdbcLogger.java:135 |http-nio-8888-exec-1|==> Parameters: 
2023-11-18 20:07:36,850|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==    Columns: uid, uname, password, gender, phone, email, address
2023-11-18 20:07:36,850|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 3, bb, 123, null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 7, L, 123, 男, 12345678900, 123@qq.com, 我家
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 8, 123, 213, 男, 321, 213, 3
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 11, , , null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 12, 111, 222, null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 13, 333, 333, null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 14, 666, 666, null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 15, 777, 777, null, null, null, null
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 16, 16, 16, 女, 1340, 11, 16道路
2023-11-18 20:07:36,851|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 17, 17, 17, 男, 111, 11, 111
2023-11-18 20:07:36,852|TRACE|     BaseJdbcLogger.java:141 |http-nio-8888-exec-1|<==        Row: 18, 19, 20, 222, 222, 222, 222
2023-11-18 20:07:36,852|DEBUG|     BaseJdbcLogger.java:135 |http-nio-8888-exec-1|<==      Total: 11
2023-11-18 20:07:36,853|DEBUG|             Logger.java:49  |http-nio-8888-exec-1|Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@15ef80e1]
2023-11-18 20:07:36,861|DEBUG|AbstractMessageConverterMethodProcessor.java:275 |http-nio-8888-exec-1|Using 'application/json;q=0.8', given [text/html, application/xhtml+xml, image/avif, image/webp, image/apng, application/xml;q=0.9, */*;q=0.8, application/signed-exchange;v=b3;q=0.7] and supported [application/json, application/*+json]
2023-11-18 20:07:36,863|TRACE|     LogFormatUtils.java:117 |http-nio-8888-exec-1|Writing [[User(uid=3, uname=bb, password=123, gender=null, phone=null, email=null, address=null, age=null), User(uid=7, uname=L, password=123, gender=男, phone=12345678900, email=123@qq.com, address=我家, age=null), User(uid=8, uname=123, password=213, gender=男, phone=321, email=213, address=3, age=null), User(uid=11, uname=, password=, gender=null, phone=null, email=null, address=null, age=null), User(uid=12, uname=111, password=222, gender=null, phone=null, email=null, address=null, age=null), User(uid=13, uname=333, password=333, gender=null, phone=null, email=null, address=null, age=null), User(uid=14, uname=666, password=666, gender=null, phone=null, email=null, address=null, age=null), User(uid=15, uname=777, password=777, gender=null, phone=null, email=null, address=null, age=null), User(uid=16, uname=16, password=16, gender=女, phone=1340, email=11, address=16道路, age=null), User(uid=17, uname=17, password=17, gender=男, phone=111, email=11, address=111, age=null), User(uid=18, uname=19, password=20, gender=222, phone=222, email=222, address=222, age=null)]]
2023-11-18 20:07:36,881|TRACE|WebContentGenerator.java:405 |http-nio-8888-exec-1|Applying default cacheSeconds=-1
2023-11-18 20:07:36,881|TRACE|  DispatcherServlet.java:1166|http-nio-8888-exec-1|No view rendering, null ModelAndView returned.
```

这里调用了三次findAll方法，日志输出只查询了一次数据库。<br>
去掉事务注解，结果如下：

![mybatis_cache_1](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-18-mybatis_source_code_reading_5/mybatis_cache_1.png)

### 1.2 源码解读

若设置cache-enabled为false，则创建Executor是SimpleExecutor：

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

从DefaultSqlSession#selectList方法进入BaseExecutor#query方法。

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)
      throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }
```

这里先生成了缓存key，CacheKey源码如下：
```java
public class CacheKey implements Cloneable, Serializable {

    private static final int DEFAULT_MULTIPLIER = 37;
    private static final int DEFAULT_HASHCODE = 17;
  
    private final int multiplier;
    private int hashcode;
    private long checksum;
    private int count;
    private List<Object> updateList;
  
    public CacheKey() {
      this.hashcode = DEFAULT_HASHCODE;
      this.multiplier = DEFAULT_MULTIPLIER;
      this.count = 0;
      this.updateList = new ArrayList<>();
    }
  
    public CacheKey(Object[] objects) {
      this();
      updateAll(objects);
    }
  
    public int getUpdateCount() {
      return updateList.size();
    }
  
    public void update(Object object) {
      int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
  
      count++;
      checksum += baseHashCode;
      baseHashCode *= count;
  
      hashcode = multiplier * hashcode + baseHashCode;
  
      updateList.add(object);
    }
  
    
    
  
    @Override
    public int hashCode() {
      return hashcode;
    }
  
    @Override
    public String toString() {
      StringJoiner returnValue = new StringJoiner(":");
      returnValue.add(String.valueOf(hashcode));
      returnValue.add(String.valueOf(checksum));
      updateList.stream().map(ArrayUtil::toString).forEach(returnValue::add);
      return returnValue.toString();
    }
    
  }
  
```
生成缓存Key的方法如下：
```java
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

可以看到这里的Key是MappedStatement的ID、Offset、Limit、Parameters、Environment组合的信息作为缓存的键。

再来到query方法。

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
      CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
                            //清除一级缓存
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
                            //查询一级缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
                            //一级缓存
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
                            //查询数据库
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
                            //清除一级缓存
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }

```
这里是先查询一级缓存再去查询数据库。<br>
在查询数据库的方法内，有存入一级缓存。
```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds,
      ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
                    //存入一级缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
这样在开启事务时候，同一个事物里面的DefaulSqlSession是同一个对象，Executor对象也是同一个，Executor.localCache也是同一个Map，所以多次相同查询会命中同一个key，就只有第一次会查询数据库，接下来的就只查询localCache这个本地缓存。

localCache的类型是PerpetualCache，其内部是使用一个简单HashMap作为缓存的。
```java
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();
    //......
}
```

insert、delete、update都会清理一级缓存：
```java
  @Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
```

```java
 @Override
  public int delete(String statement) {
    return update(statement, null);
  }

```

```java
@Override
  public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }
```
### 1.3 总结

Mybatis一级缓存的作用域是Session级别，内部是使用一个简单的HashMap实现。Mybatis一级缓存默认启用的，没有配置开关的参数，可以设置作用域为SESSION或者STATEMENT。<br>
insert、delete、update都会清理一级缓存。

## 二 、二级缓存

虽然cache-enabled全局开关是默认开启的，但是二级缓存必须指定Cache的实现类。

```java
configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
```

这里获得指定的cache：

```java
Cache cache = ms.getCache();
    if (cache != null) {
        //......
        }
    //.....
```
### 2.1 简单使用
这里使用redis作为mybatis的二级缓存为例。<br>

1. 安装redis

因为redis默认不支持windows环境，要按照windows版本可以参考[https://github.com/redis-windows/redis-windows/](https://github.com/redis-windows/redis-windows/)这个项目。

笔者这里就使用docker简单安装最新版本的redis，如下。

```shell
docker pull redis

docker run --name redis -it -p 6379:6379 -v /data/redis-data  redis --requirepass "123456"

docker exec -it redis redis-cli -a "123456"
```

使用redis-cli登录简单测试一下 (<strong>PS:keys * 慎用，读者使用后果自负</strong>)：

```shell
127.0.0.1:6379> keys *
(empty array)
127.0.0.1:6379> set 'mykey' '12234'
OK
127.0.0.1:6379> get 'mykey'
"12234"
127.0.0.1:6379>

```

2. 引入redis模块和配置redis

这里读者使用的是spring-boot-starter-data-redis模块：
```xml
   <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>io.lettuce</groupId>
                    <artifactId>lettuce-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

redis配置如下：
```yaml
spring:
  data:
    redis:
      database: 0
      host: localhost
      port: 6379
      password: 123456
      timeout: 30000
      username:
      connection-timeout: 30000
      jedis:
        pool:
          max-active: 8
          max-idle: 8
          max-wait: -1ms
          min-idle: 0

```

然后实现一个简单RedisTemplate类：
```java
/**
 * redis配置类
 * 见 JedisConnectionConfiguration、RedisAutoConfiguration等自动配置类
 */
@Configuration
public class RedisConfig {
    /**
     * 简单定制RedisTemplate
     *
     * @param redisConnectionFactory
     * @return
     */
    @Bean(name = "myRedisTemplate")
    public RedisTemplate<String, Object> myRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(om, Object.class);
        // 设置值（value）的序列化采用Jackson2JsonRedisSerializer。
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        // 设置键（key）的序列化采用StringRedisSerializer。
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
}

```
部分配置请阅读RedisAutoConfiguration等自动配置类。这样就可以使用redis功能了。

3. 使用redis作为mybatis二级缓存

使用Mybatis的二级缓存首先要打开全局的开关：

```yaml
mybatis:
  mapper-locations: classpath:mappers/*.xml
  configuration:
    cache-enabled: true
```

然后要指定Cache的实现类：

```java
@Repository
@CacheNamespace(implementation = com.homura.myspringboot.cache.MybatisRedisCacheImpl.class, eviction = com.homura.myspringboot.cache.MybatisRedisCacheImpl.class)
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
        //.....
}
```
cache实现类：
```java
/**
 * 使用redis简单实现mybatis的二级缓存
 * Created by homura on 2023/11/18.
 */
public class MybatisRedisCacheImpl implements Cache {
    private static final Logger LOGGER = LoggerFactory.getLogger(MybatisRedisCacheImpl.class);
    private static final ReadWriteLock LOCK = new ReentrantReadWriteLock();
    private static final int EXPIRE_TIME_IN_MINUTES = 30;
    private final String id;
    private static RedisTemplate<String, Object> redisTemplate;
    /**
     * 注意这里注入是静态变量
     *
     * @param redisTemplate
     */
    public static void setRedisTemplate(RedisTemplate<String, Object> redisTemplate) {
        MybatisRedisCacheImpl.redisTemplate = redisTemplate;
    }
    public MybatisRedisCacheImpl( String id) {
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public void putObject(Object key, Object value) {
        ValueOperations<String, Object> opsForValue = redisTemplate.opsForValue();
        opsForValue.set(key.toString(), value, EXPIRE_TIME_IN_MINUTES, TimeUnit.MINUTES);
        LOGGER.info("Put query result to redis");
    }

    @Override
    public Object getObject(Object key) {
        ValueOperations<String, Object> opsForValue = redisTemplate.opsForValue();
        LOGGER.info("Get cached query result from redis");
        return opsForValue.get(key.toString());
    }

    @Override
    public Object removeObject(Object key) {
        redisTemplate.delete(key.toString());
        LOGGER.info("Remove cached query result from redis");
        return null;
    }

    @Override
    public void clear() {
        //NOP
    }

    @Override
    public int getSize() {
        return 0;
    }

    @Override
    public ReadWriteLock getReadWriteLock() {
        return LOCK;
    }


}
```

注入Spring的RedisTemplate。

```java
@Configuration
public class MybatisCacheConfig {

    @Autowired
    public void setRedisTemplate(@Autowired @Qualifier("myRedisTemplate") RedisTemplate<String, Object> redisTemplate) {
        MybatisRedisCacheImpl.setRedisTemplate(redisTemplate);
    }
}
```
读者可以参考mybatis官方的mybatis-redis项目来实现redis作为mybatis的二级缓存功能，目前在官方参考只有beta版本，请谨慎选用。

然后启动应用开始测试，再读取redis的缓存的数据。
```shell
127.0.0.1:6379> keys *
1) "1270070082:3415082458:com.homura.myspringboot.dao.UserMapper.findAll:0:2147483647:select * from user:SqlSessionFactoryBean"
2) "mykey"
127.0.0.1:6379> get '1270070082:3415082458:com.homura.myspringboot.dao.UserMapper.findAll:0:2147483647:select * from user:SqlSessionFactoryBean'
"[{\"uid\":3,\"uname\":\"bb\",\"password\":\"123\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":7,\"uname\":\"L\",\"password\":\"123\",\"gender\":\"\xe7\x94\xb7\",\"phone\":\"12345678900\",\"email\":\"123@qq.com\",\"address\":\"\xe6\x88\x91\xe5\xae\xb6\",\"age\":null},{\"uid\":8,\"uname\":\"123\",\"password\":\"213\",\"gender\":\"\xe7\x94\xb7\",\"phone\":\"321\",\"email\":\"213\",\"address\":\"3\",\"age\":null},{\"uid\":11,\"uname\":\"\",\"password\":\"\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":12,\"uname\":\"111\",\"password\":\"222\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":13,\"uname\":\"333\",\"password\":\"333\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":14,\"uname\":\"666\",\"password\":\"666\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":15,\"uname\":\"777\",\"password\":\"777\",\"gender\":null,\"phone\":null,\"email\":null,\"address\":null,\"age\":null},{\"uid\":16,\"uname\":\"16\",\"password\":\"16\",\"gender\":\"\xe5\xa5\xb3\",\"phone\":\"1340\",\"email\":\"11\",\"address\":\"16\xe9\x81\x93\xe8\xb7\xaf\",\"age\":null},{\"uid\":17,\"uname\":\"17\",\"password\":\"17\",\"gender\":\"\xe7\x94\xb7\",\"phone\":\"111\",\"email\":\"11\",\"address\":\"111\",\"age\":null},{\"uid\":18,\"uname\":\"19\",\"password\":\"20\",\"gender\":\"222\",\"phone\":\"222\",\"email\":\"222\",\"address\":\"222\",\"age\":null}]"
127.0.0.1:6379>

```
这里使用Json序列化缓存值。多次请求该方法，观察日志只有一次查询数据库，只查询了Redis缓存。

### 2.2 源码解读

前面解读了如果cache-enabled设置为true，则创建的Executor是CachingExecutor。我们来到CachingExecutor#query方法。
```java
 @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler)
      throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
这里cacheKey是一样的。
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler,
      CacheKey key, BoundSql boundSql) throws SQLException {
      Cache cache = ms.getCache();
                         //如果指定了Cache实现类才会使用二级缓存，如上面的MybatisRedisCacheImpl
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")  
                        //获取二级缓存
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
                        //delega.query方法就是上文的BaseExecutor#query
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                        //存入二级缓存
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
可以看到这里二级缓存是在一级缓存外层(之前)生效的，也就是mybatis查询数据的顺序是：1.二级缓存；2.一级缓存；3.数据库。<br>
二级缓存委派给TransactionalCacheManager去处理：
```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }

  private TransactionalCache getTransactionalCache(Cache cache) {
    return MapUtil.computeIfAbsent(transactionalCaches, cache, TransactionalCache::new);
  }

}
```

然后来到
```java
public class TransactionalCache implements Cache {
    private final Cache delegate;

    @Override
    public Object getObject(Object key) {
        // issue #116
        Object object = delegate.getObject(key);
        if (object == null) {
            entriesMissedInCache.add(key);
        }
        // issue #146
        if (clearOnCommit) {
            return null;
        }
        return object;
    }
    //....
}
```
这里的TransactionalCache.delegate最终使用MybatisRedisCacheImpl去执行，如下：

![TransactionalCache.delegate](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-18-mybatis_source_code_reading_5/TransactionalCache.delegate.png)

### 2.3 总结

二级缓存需要显示开启cache-enabled和指定cache实现类。<br>
二级缓存基于Namespace，也就是作用域是Mapper级别。<br>
Mybatis的二级缓存应该慎重选用，在单机或者分布式缓存都会引发一致性的问题，需要慎重考虑，笔者的建议是不要使用Mybatis的二级缓存。


总结的说，mybatis查询数据的顺序是：1.二级缓存；2.一级缓存；3.数据库。<br>
这里使用这个图总结一下Mybatis的一级和二级缓存，图片来源[美团技术团队](https://tech.meituan.com/2018/01/19/mybatis-cache.html)。

![mybatis-cache](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-18-mybatis_source_code_reading_5/mybatis-cache.png)

## 三、参考材料

1.https://mybatis.org/mybatis-3/zh/ <br>
2.Mybatis源码(版本3.5.13) <br>
3.https://tech.meituan.com/2018/01/19/mybatis-cache.html
