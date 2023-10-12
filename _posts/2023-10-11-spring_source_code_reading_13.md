---
layout: post
title: "Spring源码阅读十二:JDK动态代理的AOP代理对象的创建流程"
date: 2023-10-10
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十二篇文章，本文介绍JDK动态代理创建的AOP对象的过程。<br>
上一篇文章介绍AopProxy有JdkDynamicAopProxy、ObjenesisCglibAopProxy（CglibAopProxy的子类）两个，是创建代理对象的委派类。<br>
AopProxy.getProxy就是使用Jdk动态代理或者Cglib动态代理生成实际的代理对象的方法。本文介绍JdkDynamicAopProxy创建代理对象的过程。<br>
依旧是先看下JdkDynamicAopProxy的简单使用，

# 一、Jdk动态动态的简单使用

服务接口：
```java
package com.homura.aopproxy.cglib;

public interface TestService {
    void test();
}

```
服务接口实现类:
```java
package com.homura.aopproxy.cglib;

import org.slf4j.Logger;

public class TestServiceImpl implements TestService {
    private static final Logger LOGGER = org.slf4j.LoggerFactory.getLogger(TestServiceImpl.class);

    @Override
    public void test() {
        LOGGER.info("test");
    }
}

```
代理对象：
```java
package com.homura.aopproxy.jdk;


import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class TestProxy implements InvocationHandler {
    private static final Logger LOGGER = LoggerFactory.getLogger(TestProxy.class);
    /**
     * 目标对象
     */
    private Object target;

    public Object newProxyInstance(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }


    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (!method.getName().contains("test")) {
            return method.invoke(target, args);
        }
        LOGGER.info("调用前:[{}]-[{}]", proxy, method);
        Object result = method.invoke(target, args);
        LOGGER.info("调用后:[{}]-[{}]", proxy, method);
        return result;
    }
}

```
测试启动类：
```java
package com.homura.aopproxy.cglib;

public class TestStarter {
    public static void main(String[] args) {
        TestProxy testProxy = new TestProxy();
        TestService testService = (TestService) testProxy.newProxyInstance(new TestServiceImpl());
        testService.test();
    }
}

```

测试结果如下:
```text
2023-10-10 20:29:32,857|INFO |          TestProxy.java:29  |main|调用前:[com.homura.aopproxy.jdk.TestServiceImpl@402c4085]-[public abstract void com.homura.aopproxy.jdk.TestService.test()]
2023-10-10 20:29:32,863|INFO |    TestServiceImpl.java:10  |main|test
2023-10-10 20:29:32,863|INFO |          TestProxy.java:31  |main|调用后:[com.homura.aopproxy.jdk.TestServiceImpl@402c4085]-[public abstract void com.homura.aopproxy.jdk.TestService.test()]

```
可以看到在代码层面基本的操作就是实现InvocationHandler，调用Proxy类来创建代理对象。
我们再继续阅读Spring使用Jdk动态代理创建代理对象的源码。

# 二、Jdk动态代理的创建代理对象过程

进入JdkDynamicAopProxy的getProxy方法：
```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
        logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
        }
        return Proxy.newProxyInstance(determineClassLoader(classLoader), this.proxiedInterfaces, this);
        }
```
这里依旧是Proxy.newProxyInstance创建代理对象。
JdkDynamicAopProxy自己实现了InvocationHandler，我们看下invoke方法:
```java

	/**
	 * Implementation of {@code InvocationHandler.invoke}.
	 * <p>Callers will see exactly the exception thrown by the target,
	 * unless a hook method throws an exception.
	 */
	@Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// Get the interception chain for this method.
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fall back on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// We need to create a method invocation...
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}


```
基本和上面的示例一样。我们继续阅读这行代码:
```java
	            // We need to create a method invocation...
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
```
然后进入ReflectiveMethodInvocation的proceed方法:
```java
@Override
	@Nullable
	public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.matcher().matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor().invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```
这里是调用目标类的方法，JDK动态代理的源码解读到这，jdk自身的源码暂不做解读。

# 三、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)
