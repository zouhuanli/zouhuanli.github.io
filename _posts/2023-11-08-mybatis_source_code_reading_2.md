---
layout: post
title: "Mybatis源码解读二:Mybatis初始化过程"
date: 2023-11-13
tags: [ mybatis ]
comments: true
author: zouhuanli
---

本文是mybatis源码阅读计划的第而篇文章,本文简单介绍一下Mybatis的初始化过程，也就是解读一下Configuration和SqlSessionFactory的启动过程。<br>

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

## 一、概述

Configuration类是是mybatis的核心配置类，保存了mybatis所有配置信息。既包括mybatis-config.xml和配置文件的配置如cacheEnabled，也包括解析之后生成的配置如mapperRegistry、interceptorChain。
"每个基于MyBatis的应用都是以一个SqlSessionFactory的实例为核心的。SqlSessionFactory的实例可以通过SqlSessionFactoryBuilder获得。而SqlSessionFactoryBuilder则可以从XML配置文件或一个预先配置的Configuration实例来构建出SqlSessionFactory实例"。
SqlSessionFactory可以由SqlSessionFactoryBuilder或者SqlSessionFactoryBean创建。
SqlSessionFactoryBean#getObject方法。

```java
  @Override
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }

    return this.sqlSessionFactory;
  }
```

SqlSessionFactory顾名思义，就是用来创建SqlSession对象的。SqlSession提供CRUD、提交、回滚、获取Mapper等顶层API。
SqlSessionFactory一般是一个数据源创建一次，不需要多次创建。而SqlSession的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。也就表明SqlSession应该随时创建后就销毁。
但SqlSessionTemplate实现了SqlSession，其被配置为了单例对象。SqlSessionTemplate并没有共用一个SqlSession，而是通过SqlSessionInterceptor创建代理对象每次执行Mapper的SQL方法时都创建一下新的SqlSession。
SqlSessionInterceptor如下：
```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      }  finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

## 二、SqlSessionFactory创建过程

## 三、Configuration的创建过程

### 3.1 基本配置解析

### 3.2 XML配置解析

### 3.3 XMLMapper解析

## 四、SqlSessionTemplate解读

## 五、参考材料

1.https://mybatis.org/mybatis-3/zh/ <br>
2.Mybatis源码(版本3.5.13) <br>
