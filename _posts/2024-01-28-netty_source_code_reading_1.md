---
layout: post
title: "Netty源码解读:总览和系列计划"
date: 2024-01-28
tags: [ Netty ]
comments: true
author: zouhuanli
---

本系列是新开的解读Netty源码的文章系列。

要解读Netty源码需要做比较多的前期铺垫内容，主要是Linux网络、Java NIO的内容，特别是Reactor模型,这些基础内容对理解其他网络框架也是有所帮助的。

本系列文章在网络模型、Java NIO的基础之上，打算按照两条脉络来解读Netty的使用和源码分析：<br>
1. 组件的创建、启动、初始化和启动。<br>
2. 网络请求的收发处理过程。<br>

本系列文章主要参考资料有：<br>
1. 《深入理解Linux网络技术内幕》<br>
2. 《计算机网络：自顶向下方法》<br>
3. 《TCP/IP协议详解》<br>
4. 《HTTP权威指南》<br>
5. 《Netty实战》<br>

Netty的stable版本是4.1，本系列主要以4.1的版本作为解读源码的版本。

TODO~









