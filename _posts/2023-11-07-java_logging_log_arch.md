---
layout: post
title: "Java日志体系介绍"
date: 2023-11-07
tags: [ Java日志 ]
comments: true
author: zouhuanli
---

# 一、契机

事情的起因是之前为解决log4j漏洞，需要把一个比较老旧的系统升级为较新版本的log4j2，期间花费了一下时力。所以干脆写一篇文章纪录一下Java日志的发展历史和各个日志产品之间的关系。<br>
首先解释一些名称。<br>
1.日志产品：<br>
JUL:Java Util Logging,Java源码原生自带的日志产品，也叫JDK Logging。<br>
Log4j1:使用较广的Java日志产品。<br>
logback:slf4j的日志作者创建的日志框架，SpringBoot的默认日志框架。<br>
Log4j2:Log4j1的升级版，Apache吸取上面日志的产品优点然后重写Log4j1，在性能上有较大提升。<strong>
注意Log4j2和Log4j1不兼容，不是简单的版本号升级的区别,应该视为一款新的日志实现</strong>。<br>
2.日志API(门面):<br>
JCL:common-logging,Apache的日志API，只是API，没有日志实现，没法单独使用。Spring源码默认使用的日志API。<br>
SLF4J:Simple Logging Facade for Java，简单Java日志门面，只是API，没有日志实现，没法单独使用。<br>

Ceki Gülcü是log4j1,logback,SLF4J的作者，可以说是以一人之力支撑了Java日志的发展。<br>
本文不会讲解如何具体配置和使用logback或者log4j2。<br>
本文的源码地址是[https://github.com/zouhuanli/loggingdemo](https://github.com/zouhuanli/loggingdemo)。

# 二、Java日志历史

## 2.1  System.out.print

回归到最原始纪录日志的方式: System.out.print。<br>
这种控制台打印方式最原始最简单，用于简单调试是可行的，不会应用于生产代码和生产环境。

## 2.2 Log4j1

log4j1是1996年出现的日志产品，很长时间是Java日志事实的标准。log4j1在2015年停止维护更新，所以现在不应该再使用了log4j1

## 2.3 JUL

JUL是JDK1.4引入的官方的自带日志产品，不需要额外依赖,整体不如log4j1，使用的比较少。

## 2.4 JCL（Jakarta Commons Logging）

Apache推出的日志门面，只有API，没有实现。是Spring源码默认使用的日志API。

## 2.5  SLF4j

SLF4j是log4j主要作者Ceki开发的日志接口，只有API接口，没有具体的实现。需要通过各种桥连包(适配器)绑定具体的日志实现产品。<br>
比如SLF4j,log4j1,slf4j-log4j12组合使用,即可以通过SLF4J的API绑定log4j1的实现，在代码中使用SLF4J输出日志。

## 2.6 Logback

Logback是Ceki离开Apache后一个人开发的日志产品实现，原生实现了SLF4j，和SLF4J结合不需要桥连包。<br>
Logback是Log4j1的改进和升级版本，是SpringBoot默认集成的日志框架。使用logback一般引入slf4j-api,logback-core,logback-classic

## 2.7 Log4j2

Apache在logback之后开发了log4j2，是log4j1的升级版本，但是！和log4j1完全不兼容。<br>
使用log4j2一般引入log4j-core和log4j-api,如果使用slf4j-api作为门面，需要引入log4j-slf4j-impl作为log4j2自带的实现slf4j-api的桥连包。<br>
如:

```xml

<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>${log4j2.version}</version>
</dependency>
<dependency>
<groupId>org.apache.logging.log4j</groupId>
<artifactId>log4j-api</artifactId>
<version>${log4j2.version}</version>
</dependency>

<dependency>
<groupId>org.slf4j</groupId>
<artifactId>slf4j-api</artifactId>
<version>${slf4j.version}</version>
</dependency>
<dependency>
<groupId>org.apache.logging.log4j</groupId>
<artifactId>log4j-slf4j2-impl</artifactId>
<version>${log4j2.version}</version>
</dependency>
```

# 三、Java日志各种产品的关系

在slf4j出现之后，出现了各种桥连包，将具体的日志产品桥连到slf4-api，从而通过使用slf4j-api来使用具体的日志产品。<br>
slf4-api本身只有接口，必须绑定具体日志实现才能使用，绑定相关的桥连包如下：

![concrete-bindings](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-07-java_logging_log_arch/concrete-bindings.png)

桥连包极大的方便了开发者自由切换日志产品，更换日志产品时候只需要切换到其他日志实现，添加对应的桥连包即可。<br>
各种桥连包和日志产品的关系如下图：

![slf4j-bridge](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-07-java_logging_log_arch/slf4j-bridge.png)

我们当然不要全部记住这些包，我们要注意区分日志实现、日志门面、日志桥连包即可。<br>
注意谨慎选用桥连包，一定要去官方网站查询使用说明，<strong>循环依赖会导致无法输出日志</strong>。<br>
例如：

```text
Please note that jcl-over-slf4j.jar and slf4j-jcl.jar cannot be deployed at the same time. 
The former jar file will cause JCL to delegate the choice of the logging system to SLF4J and the latter jar file will cause SLF4J to delegate the choice of the logging system to JCL, resulting in an infinite loop.
```

请注意，jcl-over-slf4j.jar和
slf4j-jcl.jar不能同时部署。前一个jar文件将导致JCL将日志系统的选择委托给SLF4J，而后一个jar文件将导致SLF4J将日志系统的选择委托给JCL，从而导致无限循环。再这种情况下就会导致无法输出日志。

# 四、具体案例

假设有一个项目使用了下列的日志包（仅供演示,请读者根据自己的具体要求选择对应的日志包）:

```xml
 <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <project.resources.sourceEncoding>UTF-8</project.resources.sourceEncoding>
        <!-- 1.8之后的版本的slf4j有所不同，请参考官网配置-->
        <slf4j.version>1.7.30</slf4j.version>
        <log4j2.version>2.17.1</log4j2.version>
    </properties>
    <dependencies>
        <!-- log4j2 core和api-->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>${log4j2.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>${log4j2.version}</version>
        </dependency>

        <!--slf4j-api-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>

        <!--slf4j-log4j12,将slf4j-api桥连到log4j1-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>${slf4j.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--jcl-over-slf4j,使用jcl打印日志将会转接到slf4j-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <!--log4j-1.2-api，log4j1升级到log4j2的包， 参考：https://logging.apache.org/log4j/2.x/manual/migration.html -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-1.2-api</artifactId>
            <version>${log4j2.version}</version>
        </dependency>
        <!--jul-to-slf4j,使用JUL打印日志将会转接到slf4j-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jul-to-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <!--Spring源码自带的JCL-->
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>

    </dependencies>
    
```
首先这里将JUL、JCL都桥连到了SLF4J，SLF4J由桥连到了Log4j1，而工程里面已经没有Log4j1依赖包。Log4j1通过log4j-1.2-api桥连到了Log4j2,这是一个log4j1升级到log4j2的包，由log4j2官方提供。<br>
Log4j1通过log4j-1.2-api桥连到了Log4j2,这样Log4j1升级到Log4j2就不需要修改应用的代码。<br>
最后的结果是：Log4j1，slf4j,jul,jcl都桥连到了log4j2。 简单测试如下。

1.Log4j1的测试。<br>
测试类如下：

```java
import org.apache.log4j.Logger;


public class Log4j1Test {
    //注意这个Logger类是旧的log4j1的类
    private static final Logger LOGGER = Logger.getLogger(Log4j2Test.class);

    public static void main(String[] args) {
        LOGGER.info("Hello World!" + Log4j1Test.class.getSimpleName());
    }
}
```

跟踪执行流程来到这里：

```java
    private final org.apache.logging.log4j.Logger logger;

    /**
     * Constructor used by Logger to specify a LoggerContext.
     * @param context The LoggerContext.
     * @param name The name of the Logger.
     */
    protected Category(final LoggerContext context, final String name) {
        this.logger = context.getLogger(name);
        rendererMap = ((RendererSupport) LogManager.getLoggerRepository()).getRendererMap();
    }
```
这里桥连到log4j2去处理了。<br>

2.Log4j2的测试.<br>

测试类如下：
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Log4j2Test {
    //log4j2+slf4j的Logger类
    private static final Logger LOGGER = LoggerFactory.getLogger(Log4j2Test.class);

    public static void main(String[] args) {
        LOGGER.info("Hello World!" + Log4j2Test.class.getSimpleName());
    }
}

```

在StaticLoggerBinder这里绑定了log4j1的日志实现。

```java
 private StaticLoggerBinder() {
        loggerFactory = new Log4jLoggerFactory();
        try {
            @SuppressWarnings("unused")
            Level level = Level.TRACE;
        } catch (NoSuchFieldError nsfe) {
            Util.report("This version of SLF4J requires log4j version 1.2.12 or later. See also http://www.slf4j.org/codes.html#log4j_version");
        }
    }
```

然后在Log4jLoggerFactory这里创建Logger:

```java
 public Logger getLogger(String name) {
        Logger slf4jLogger = loggerMap.get(name);
        if (slf4jLogger != null) {
            return slf4jLogger;
        } else {
            org.apache.log4j.Logger log4jLogger;
            if (name.equalsIgnoreCase(Logger.ROOT_LOGGER_NAME))
                log4jLogger = LogManager.getRootLogger();
            else
                log4jLogger = LogManager.getLogger(name);

            Logger newInstance = new Log4jLoggerAdapter(log4jLogger);
            Logger oldInstance = loggerMap.putIfAbsent(name, newInstance);
            return oldInstance == null ? newInstance : oldInstance;
        }
    }
```

最后转接到log4j2的实现，使用log4j2日志实现输出日志。<br>

为什么要使用这么多桥连包转接？<br>
假定原来此工程使用的是slf4j-api,log4j,log4j-over-slf4j三者的组合，代码只有slf4j-api的api。<br>
那么由log4j1升级到log4j2，只需要移除log4j,log4j-over-slf4j，再添加log4j2的log4j-core和log4j-api,加上log4j-slf4j-impl。<br>
这样使用slf4j-api的API的代码能够直接切换为Log4j2实现。<br>
<strong>所以要使用slf4j作为日志门面API，而不能使用具体日志实现</strong>。


# 五、使用原则

把握以下一些的原则，将帮助你理解Java整个日志体系：<br>
<strong>1.面向接口编程，使用日志门面API，而不是具体日志实现的API或者类</strong>。这样做的好处是可以很方便替换具体的日志的实现，只需要移除旧的日志实现和桥连包，加入新的日志实现和桥连包，不需要修改任何应用代码。<br>
<strong>2.只使用一款日志具体实现，推荐logback或者log4j2</strong>。<br>
<strong>3.谨慎引入桥连包，避免循环依赖导致的无限循环</strong>。<br>

# 六、参考材料

1.https://juejin.cn/post/6905026199722917902 <br>
2.https://logging.apache.org/log4j/2.x/manual/migration.html <br>
3.https://www.slf4j.org/manual.html <br>
4.https://www.cnblogs.com/lalalazar/p/15694889.html <br>