---
layout: post
title: "GitHub相关:GitHu博客无法显示图片的问题的解决方法"
date: 2023-10-18
tags: [ spring ]
comments: true
author: zouhuanli
---

由于一些懂得都懂的原因，国内上github网站经常断断续续的连接不上，而且github的图片不管一些开源项目的，还是博客的，都会无法下载直接显示"?"图标。<br>
下面是解决方法：

1.使用梯子。读者不会使用梯子的请自己去解决，笔者不提供任何关于梯子的帮助。<br>
2.登录一个查询DNS的网站。比如笔者博客的一个图片的地址'https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-19-spring_source_code_reading_18/txInfo.png',
复制域名‘raw.githubusercontent.com’，查询DNS获得IP，如下（不保证正确性，也不保证为了不会变更)：<br>

```text
185.199.110.133 raw.githubusercontent.com
185.199.108.133 raw.githubusercontent.com
185.199.111.133 raw.githubusercontent.com
185.199.109.133 raw.githubusercontent.com
```
3.将对应的IP添加到hosts文件中。

