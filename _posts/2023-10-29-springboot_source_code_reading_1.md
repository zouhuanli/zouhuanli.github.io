---
layout: post
title: "SpringBoot源码阅读一:SpringBoot整体概览"
date: 2023-10-29
tags: [ springboot ]
comments: true
author: zouhuanli
---

本文是SpringBoot源码阅读计划的第一篇文章，本文主要介绍一下SpringBoot的整体功能和使用一个示例功能开始上手使用SpringBoot和理解SpringBoot。

SpringBoot像较于Spring Framework并不是一个新的框架，而是极大的简化使用Spring框架、提供快速简单使用Spring的方式。<br>
SpringBoot提供一系列的stater组件，极大的简化配置管理和依赖管理，从而提简单快速搭建spring应用。<br>
传统的Spring应用有很重的XML配置和各种各样的配置文件，SpringBoot正是为了简化配置、快速开发而生。<br>
本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。
示例工程的JDK版本是JDK17，JDK21刚刚出来还未引入到Spring系列框架，SpringBoot版本是3.1.3，其对应的Spring
Framework版本是6.0.11。<br>
不同版本之间的差异此系列不作任何解读和比较。

# 一、SpringBoot概述

下列来自SpringBoot官方文档。
SpringBoot的主要目标是：
• 为所有的Spring开发提供一个根本性的更快、更广泛的入门体验。
• 开箱即用，但随着需求开始偏离默认值，请迅速摆脱与默认设置偏离的困境。
• 提供一系列大类项目常见的非功能特性（如嵌入式服务器、安全、度量、健康检查和外部化配置）。
• 几乎没有代码生成，也不要求XML配置。

可以看到Spring最核心的功能就是：
<strong>1.自动依赖管理</strong>。通过各系列的stater将所需依赖项统一管理。
<strong>2.自动配置</strong>。通过自动配置模块统一管理配置，这里基于约定优于配置的思想，会自动选择需要的配置。
此外增加了一些企业级的功能，如Actuator监控检测和监控。

把握这两个核心功能有助于我们简化理解SpringBoot。

# 二、示例工程

笔者这里的示例工程的版本是比较高的，不同版本之间的SpringBoot有所差异，这里不做过多比较和解读。

工程依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.homura</groupId>
    <artifactId>MySpringBoot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>MySpringBoot</name>
    <description>MySpringBoot</description>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <spring.boot.version>3.1.3</spring.boot.version>
        <guawa.version>32.1.2-jre</guawa.version>
        <mysql-connector.version>8.0.33</mysql-connector.version>
        <mybatis-spring-boot-starter.version>3.0.2</mybatis-spring-boot-starter.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring.boot.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>${guawa.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
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
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>


    </build>

</project>

```

log4j2配置,log4j2是笔者引入替换logback的，若使用默认的logback请参照对应配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Properties>
        <Property name="APP">myspringboot</Property>
        <Property name="LOG_HOME">./logs</Property>
        <Property name="LOG_FILE">${APP}_console.log</Property>
        <Property name="ERROR_LOG">error.log</Property>
        <Property name="LOG_LEVEL">DEBUG</Property>
        <Property name="LOG_PATTERN">%d|%-5p|%24F:%-4L|%t|%m%n</Property>
    </Properties>
    <Appenders>
        <Console name="Console">
            <PatternLayout>
                <Pattern>${LOG_PATTERN}</Pattern>
            </PatternLayout>
            <Filters>
                <ThresholdFilter level="DEBUG" onMatch="ACCEPT" onMisMatch="NEUTRAL"/>
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMisMatch="NEUTRAL"/>
            </Filters>
        </Console>

        <RollingFile name="Root" fileName="${LOG_HOME}/${LOG_FILE}"
                     filePattern="${LOG_HOME}/${LOG_FILE}.%d{yyyyMMdd}.%i">
            <PatternLayout>
                <Pattern>${LOG_PATTERN}</Pattern>
            </PatternLayout>
            <Filters>
                <ThresholdFilter level="${LOG_LEVEL}"/>
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMisMatch="NEUTRAL"/>
            </Filters>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="200MB"/>
            </Policies>
        </RollingFile>

        <RollingFile name="ErrorLog" fileName="${LOG_HOME}/error.log"
                     filePattern="${LOG_HOME}/error.log.%d{yyyyMMdd}.%i">
            <PatternLayout>
                <Pattern>${LOG_PATTERN}</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="${LOG_LEVEL}">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="Root"/>
            <AppenderRef ref="ErrorLog" level="ERROR"/>
        </Root>
    </Loggers>


</Configuration>
```

应用主配置，application.yml。

```yaml
server:
  port: 8888
spring:
  profiles:
    active: db
```

数据库配置,application-db.yml.

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/sakila?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowPublicKeyRetrieval=true
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: sakila
    password: sakila
    type: com.zaxxer.hikari.HikariDataSource
    hikari:
      pool-name: MyBootHikariCP
      connection-test-query: SELECT 1
      validation-timeout: 3000
      read-only: false
      connection-timeout: 60000
      idle-timeout: 60000
      max-lifetime: 60000
      maximum-pool-size: 10
      auto-commit: true
      minimum-idle: 1
mybatis:
  mapper-locations: classpath:mappers/*.xml
  configuration:
    cache-enabled: true
    use-generated-keys: false
    default-executor-type: simple
```

应用主类：

```java

@SpringBootApplication
public class MySpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApplication.class, args);
    }

```

mybatis配置:

```java

/**
 * 注解方式mybatis配置类
 */
@Configuration
@EnableTransactionManagement(proxyTargetClass = true, mode = AdviceMode.PROXY)
@MapperScan(basePackages = "com.homura.myspringboot.dao")
public class MyBatisConfig {
}
```

测试的controller：

```java

@RestController
@RequestMapping("/user")
public class UserController {
    private static final Logger LOGGER = LoggerFactory.getLogger(UserController.class);
    @Autowired
    private UserService userService;

    @RequestMapping(value = "/add", method = RequestMethod.POST, produces = "application/json")
    public ResponseResult<User> add(@Valid @RequestBody User user, BindingResult bindingResult) {
        LOGGER.info("add method:{}", user);
        if (bindingResult.hasErrors()) {
            List<ObjectError> errors = bindingResult.getAllErrors();
            errors.forEach(p -> {
                FieldError fieldError = (FieldError) p;
                LOGGER.error("Invalid Parameter : object - {},field - {},errorMessage - {}", fieldError.getObjectName(), fieldError.getField(), fieldError.getDefaultMessage());
            });
            return ResponseResult.fail("invalid parameter");
        }
        userService.add(user);
        return ResponseResult.success(user);
    }

    @GetMapping("/listAll")
    public List<User> listAll() {
        LOGGER.info("listAll method");
        return userService.listAll();
    }


    @GetMapping("/get/{uid}")
    public User getByUid(@PathVariable("uid") Integer uid) {
        LOGGER.info("listAll method");
        return userService.getByUid(uid);
    }
}
```

Service:

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

DAO接口：

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

entity类：

```java

/**
 * entity
 * @author zouhl
 */
@Data
public class User implements Serializable {
    /**
     * 用户ID
     */
    private Integer uid;
    /**
     * 用户姓名
     */
    private String uname;
    /**
     * 密码
     */
    private String password;
    /**
     * 性别
     */
    private String gender;
    /**
     * 手机号码
     */
    private String phone;
    /**
     * 邮件
     */
    private String email;
    /**
     * 地址
     */
    private String address;

    /**
     * 年龄
     */
    private Integer age;
}

```

直接执行主类,启动成功：

```text

2023-10-29 21:48:43,421|INFO |  StartupInfoLogger.java:57  |main|Started MySpringBootApplication in 1.004 seconds (process running for 1.364)
```

打开浏览器或者IDEA自带的Http Request客户端,执行测试。<br>
测试:http://localhost:8888/user/listAll
输入结果如下：
![p1_test_reuslts](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-29-springboot_source_code_reading_1/p1_test_reuslts.png)

到这里，这样一个简单的MVC工程就搭建完成了，使用spring-boot-starter-web快速搭建了WebMVC工程，使用内置的tomcat服务作为Servlet容器启动。

# 三、SpringBoot源码整体结构

从springboot的github源码地址[https://github.com/spring-projects/spring-boot.git](https://github.com/spring-projects/spring-boot.git)
下载源码。<br>
使用IDEA打开工程，等待依赖解析完成。
![modules](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-29-springboot_source_code_reading_1/modules.png)

这里简单介绍一下这些项目:<br>
test的自然就是测试相关的项目，暂时先不做过多解读。<br>
spring-boot-parent：父项目，主要是全局的依赖管理。<br>
spring-boot:boot的主项目，也是源码阅读系列的重点解读项目。<br>
spring-boot-autoconfigure：自动配置，自动配置实现的代码都在里面，是源码阅读系列的重点解读项目。<br>
spring-boot-actuator:监控和健康检测的项目。<br>
spring-boot-dependencies:推荐依赖，里面只有maven或gradle依赖，没有Java代码。里面是SpringBoot对实际开放中各个组件使用需要的依赖做的统一推荐。<br>
spring-boot-devtools：开发工具，提供比如热部署等开发者需要的工具和功能。<br>
spring-boot-docker-compose；docker部署相关的项目。<br>
spring-boot-docs；主要是项目文档了。<br>
spring-boot-starters:各种开发中实际用到的具体组件的依赖的统一管理，也是没有Java代码，只有依赖。像spring-boot-starter，spring-boot-starter-aop,
spring-boot-starter-data-jdbc,spring-boot-starter-data-redis,spring-boot-starter-web都是一系列相关组件的统一管理的依赖项，方便开发者直接使用和开发。<br>

从整个全局项目看，此系列将主要解读spring-boot、spring-boot-autoconfigure两个项目的源码。<br>
把握SpringBoot的两个核心功能自动依赖管理和自动配置，将有助于理解SpringBoot的设计。

# 四、参考材料
1.《Spring Boot Reference Documentation》(version 3.1.5)
