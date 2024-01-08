---
layout: post
title: "Tomcat源码解读八:NioEndpoint"
date: 2024-01-08
tags: [ Tomcat ]
comments: true
author: zouhuanli
---

本文简单解读Connector连接器组件内的NioEndpoint。
本系列的源码工程为：[HowTomcatWorksSourceCode](https://github.com/zouhuanli/HowTomcatWorksSourceCode.git)。

# 一、概述

上一篇文章有讲到ProtocolHandler的实现类，Tomcat 8和之后默认的协议处理器是AbstractHttp11Protocol。AbstractHttp11Protocol的构造方法创建了NioEndpoint。
```java

    public Http11NioProtocol() {
        this(new NioEndpoint());
    }


    public Http11NioProtocol(NioEndpoint endpoint) {
        super(endpoint);
    }
```
NioEndpoint是按照acceptor+poller+Workers的线程模型实现的"端点"。
```java
/**
 * NIO tailored thread pool, providing the following services:
 * <ul>
 * <li>Socket acceptor thread</li>
 * <li>Socket poller thread</li>
 * <li>Worker threads pool</li>
 * </ul>
 *
 * TODO: Consider using the virtual machine's thread pool.
 *
 * @author Mladen Turk
 * @author Remy Maucherat
 */
public class NioEndpoint extends AbstractNetworkChannelEndpoint<NioChannel,SocketChannel> {
    //......
}

```
Connector连接器相关组件工作原理如下：

1. 其内部主要由acceptor创建ServerSocket，接受来自客户端的连接，封装为Event，发布到事件队列中。<br>
2. 再由poller拉取Event将socket封装为SocketWrapper，并创建SocketProcessor(worker任务)，提交到Worker线程池。<br>
3. Worker线程池的任务由Http11Processor处理。<br>
4. Http11Processor再调用CoyoteAdapter处理。CoyoteAdapter再连接容器，将请求交给Container去处理。<br>

其中NioEndpoint是非常核心的组件，实现了acceptor+poller+Workers的线程模型，非常值得重点研究。其也体现Java同步NIO的网络编程模型。Connector整体的工作原理如下图：

![Connector](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-01-08-tomcat_source_code_reading_8/Connector.png)

图片来源:[https://www.cnblogs.com/arthinking/p/14509130.html](https://www.cnblogs.com/arthinking/p/14509130.html).

图表帮助我们记忆和理解，如果要探究这些图表/流程是从哪里来的，还是需要去研究源码。

Nio2Endpoint是异步AIO实现的网络编程模型，本系列文章不作解读，读者可以自己研究。

本文主要解读NioEndpoint的源码，同时简单解读Http11Processor这个类。


# 二、NioEndpoint

todo~

## 2.1 NioEndpoint

## 2.2 Acceptor

## 2.3 Poller

## 2.4 SocketProcessor(worker)

# 三、
# 五、参数资料

1.《深入剖析Tomcat》

