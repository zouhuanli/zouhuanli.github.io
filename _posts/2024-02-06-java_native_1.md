---
layout: post
title: "Java Native初探"
date: 2024-02-06
tags: [ Java Native ]
comments: true
author: zouhuanli
---

Java的Native化，道阻且长。

本文目录为：
- [一. Java Native概述](#一-java-native概述)
- [二. 环境准备](#二-环境准备)
- [三. 项目](#三-项目)
  - [3.1 项目源码](#31-项目源码)
  - [3.2 编译](#32-编译)
  - [3.3 运行与测试](#33-运行与测试)
- [四. 参考资料](#四-参考资料)


# 一. Java Native概述
Java的Native Image是将Java代码编译为可执行文件，在执行系统上不需要Java运行时环境，更加适配云原生的要求。

Native Image的实现需要GraalVM虚拟机+AOT编译。Ahead-Of-Time（AOT）编译是将高级 Java 代码编译成本地可执行代码的过程。曾经为了实现跨平台的JVM和ClassFile,现在又开始追求本地编译了。不过短期内实现现有框架和项目的原生镜像不可能，这是一条漫长的道路。

Native并没有那么简单实现，或者说把现有项目转换为Native Image仍然需要很多工作。主要问题是Java的动态性功能，比如反射、代理等，无法提前预知然后使用AOT编译。

原生镜像的主要优势是：
1. 即时启动。长期以来Java的应用就是因为启动要做类加载等各种操作，启动速度慢而被诟病。Native Image极大的提升了启动速度。<br>
2. 减少内存消耗。Java的应用主要是为了长驻服务器使用，对内存使用很大。IDEA 自己也是个吃内存大户。<br>
3. 轻量级镜像。这里主要指的是减少产品包的大小，方便分发，也能带来启动速度的提升。<br>

总体的说是为了更加适配云原生的要求。

Spring 6和dubbo 3已经加入native模块，开始走上了native之路。但Java代码（应用和框架）要转变为Native Image还很难，AOT要求提前预知Java代码的动态性,主要通过静态分析+hint来实现，但是并不绝对100%是完整的。而使用反射、动态代理、反序列化等动态性功能的代码随从可见，很难一步到位的迁移完成。

从好的角度看，native是Java发展至今的又一次进化。

# 二. 环境准备
项目准备如下。

主要的环境/工具：<br>
IDE:IntelliJ IDEA 2023 Ultimate <br>
JDK：Oracle GraalVM 21.0.1 <br>
Gradle：8.5 <br>
这里最重要的就是GraalVM：<br>
```text
zouhl@zouhl-mint:~/git/code/zouhuanli.github.io$ java  -version 
java version "21.0.1" 2023-10-17
Java(TM) SE Runtime Environment Oracle GraalVM 21.0.1+12.1 (build 21.0.1+12-jvmci-23.1-b19)
Java HotSpot(TM) 64-Bit Server VM Oracle GraalVM 21.0.1+12.1 (build 21.0.1+12-jvmci-23.1-b19, mixed mode, sharing)

```
需要安装好gcc，g++这些，然后安装开发工具库。<br>
```shell
sudo apt-get install build-essential libz-dev zlib1g-dev
```
为了避免各种问题，建议在MacOS或者Linux环境执行。<br>
笔者环境为：<br>
```text
Linux zouhl-mint 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

```

环境准备妥当，下面开始编写入门工程。笔者这里使用Quarkus的一个入门工程作为示例工程。

# 三. 项目

## 3.1 项目源码
项目源码地址是:[https://github.com/zouhuanli/MyQuarkus.git](https://github.com/zouhuanli/MyQuarkus.git).

这里主要是这个资源类：
```java
@Path("/hello")
public class ExampleResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello from RESTEasy Reactive";
    }
}
```

类似与Spring的Controller,请求路径为“/hello”，响应为一个简单的字符串。

## 3.2 编译
普通的正常编译后检查lib，项目使用到了netty和vertx，这里不作过多解读。

native的编译主要增加native相关参数。

执行下面命令编译为native镜像：
```shell
./gradlew build -Dquarkus.package.type=native
```

得到native镜像文件：

```text
rwxrwxr-x  1 zouhl zouhl  53M  1月  7 00:21 myquarkus-1.0-SNAPSHOT-runner*

```
这样的镜像文件大小是53M。

## 3.3 运行与测试

直接启动myquarkus-1.0-SNAPSHOT-runner:
```shell
./myquarkus-1.0-SNAPSHOT-runner 

```
控制台输出：
```text
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2024-02-06 20:33:06,991 INFO  [io.quarkus] (main) myquarkus 1.0-SNAPSHOT native (powered by Quarkus 3.6.4) started in 0.014s. Listening on: http://0.0.0.0:8080
2024-02-06 20:33:06,991 INFO  [io.quarkus] (main) Profile prod activated. 
2024-02-06 20:33:06,991 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy-reactive, smallrye-context-propagation, vertx]

```
启动成功。

浏览器输入：http://localhost:8080/hello。  页面返回：
```text
Hello from RESTEasy Reactive
```
测试成功。

到这里，一个入门级别的Java Native项目就完成了。

Native Image看起来很令人激动，但要走的路还很长。

Java的Native化，道阻且长。

# 四. 参考资料

主要是参考quarkus的官网文章

1. https://cn.quarkus.io/guides/building-native-image
2. https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html
3. https://www.graalvm.org/latest/reference-manual/native-image/







