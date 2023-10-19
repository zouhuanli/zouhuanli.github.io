---
layout: post
title: "Spring源码阅读十八:事务拦截器TransactionInterceptor"
date: 2023-10-19
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十八篇文章，本文着重解读事务拦截器TransactionInterceptor。<br>
上一篇已经解读了事务属性和事务切面Advisor(Pointcut+Advice)对象的解析和注册流程，本篇开始解读事务拦截器TransactionInterceptor对普通的数
据库操作语句方法的增强处理,TransactionInterceptor是事务处理的核心类。

本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、TransactionInterceptor事务拦截器

先看下类关系图

![TransactionInterceptor](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-19-spring_source_code_reading_18/TransactionInterceptor.png)

可以看到其顶层接口是Advice和Aware。
我们看下最核心的invoke方法。

```java
@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
			@Override
			@Nullable
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
			@Override
			public Object getTarget() {
				return invocation.getThis();
			}
			@Override
			public Object[] getArguments() {
				return invocation.getArguments();
			}
		});
	}
```

这里直接调用了invokeWithinTransaction方法，我们进入这个方法。
```java
@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
                        //获得事务属性
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
                        //获得事物管理器
		final TransactionManager tm = determineTransactionManager(txAttr);
                        //reactive栈的事物切面处理，这里先忽略
		if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
			boolean isSuspendingFunction = KotlinDetector.isSuspendingFunction(method);
			boolean hasSuspendingFlowReturnType = isSuspendingFunction &&
					COROUTINES_FLOW_CLASS_NAME.equals(new MethodParameter(method, -1).getParameterType().getName());
			if (isSuspendingFunction && !(invocation instanceof CoroutinesInvocationCallback)) {
				throw new IllegalStateException("Coroutines invocation not supported: " + method);
			}
			CoroutinesInvocationCallback corInv = (isSuspendingFunction ? (CoroutinesInvocationCallback) invocation : null);

			ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
				Class<?> reactiveType =
						(isSuspendingFunction ? (hasSuspendingFlowReturnType ? Flux.class : Mono.class) : method.getReturnType());
				ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(reactiveType);
				if (adapter == null) {
					throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type: " +
							method.getReturnType());
				}
				return new ReactiveTransactionSupport(adapter);
			});

			InvocationCallback callback = invocation;
			if (corInv != null) {
				callback = () -> KotlinDelegate.invokeSuspendingFunction(method, corInv);
			}
			Object result = txSupport.invokeWithinTransaction(method, targetClass, callback, txAttr, rtm);
			if (corInv != null) {
				Publisher<?> pr = (Publisher<?>) result;
				return (hasSuspendingFlowReturnType ? KotlinDelegate.asFlow(pr) :
						KotlinDelegate.awaitSingleOrNull(pr, corInv.getContinuation()));
			}
			return result;
		}

		PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager cpptm)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
                        //1.创建新事务或者加入已有事务
			TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
                        //2.执行目标方法或者拦截器链路的下一个拦截器
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                    //3.异常处理：事务回滚，或者提交
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                //清理资源
				cleanupTransactionInfo(txInfo);
			}

			if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
				// Set rollback-only in case of Vavr failure matching our rollback rules...
				TransactionStatus status = txInfo.getTransactionStatus();
				if (status != null && txAttr != null) {
					retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
				}
			}
                 //4.提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
                        //CallbackPreferringPlatformTransactionManager，我们配置的是DataSourceTransactionManager。这里也先忽略
		else {
			Object result;
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				result = cpptm.execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
					try {
						Object retVal = invocation.proceedWithInvocation();
						if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
							// Set rollback-only in case of Vavr failure matching our rollback rules...
							retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
						}
						return retVal;
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException runtimeException) {
								throw runtimeException;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}

			// Check result state: It might indicate a Throwable to rethrow.
			if (throwableHolder.throwable != null) {
				throw throwableHolder.throwable;
			}
			return result;
		}
	}

```
这里就是Spring事务切面处理的逻辑了，这是一个环绕通知，在目标方法前后都有增强处理逻辑。笔者这里将上面的invoke再做一下删减,得到源码如下。

```java
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
                        //获得事务属性
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
                        //获得事务管理器
		final TransactionManager tm = determineTransactionManager(txAttr);
		PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager cpptm)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
                    //创建事务或者加入事务
			TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
			Object retVal;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
                    //目标方法或者拦截器链的下一个拦截器
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                    //回滚或提交
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                //清理事务信息
				cleanupTransactionInfo(txInfo);
			}
                //事务提交
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

	}
```
这样精简一下，很容易阅读这个事务处理的方法了。<br>

## 1.获得事务属性

获取事务属性这里不做解读，见上篇文章，这里的txAttr是RuleBasedTransactionAttribute，包含了com.homura.service.UserServiceImpl
.addUser这个方法的的事务配置信息。这里开始解读其他的流程。

## 2.获取事务管理器determineTransactionManager

我们看下这个方法

```java
@Nullable
	protected TransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
		// Do not attempt to lookup tx manager if no tx attributes are set
		if (txAttr == null || this.beanFactory == null) {
			return getTransactionManager();
		}

		String qualifier = txAttr.getQualifier();
		if (StringUtils.hasText(qualifier)) {
			return determineQualifiedTransactionManager(this.beanFactory, qualifier);
		}
		else if (StringUtils.hasText(this.transactionManagerBeanName)) {
			return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
		}
		else {
			TransactionManager defaultTransactionManager = getTransactionManager();
			if (defaultTransactionManager == null) {
				defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
				if (defaultTransactionManager == null) {
					defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
					this.transactionManagerCache.putIfAbsent(
							DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
				}
			}
			return defaultTransactionManager;
		}
	}

```

这里transactionManagerBeanName由设置值，在之前的AnnotationDrivenBeanDefinitionParser注册三个bean的时候：

```java

				// Create the TransactionInterceptor definition.
				RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
				interceptorDef.setSource(eleSource);
				interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				registerTransactionManager(element, interceptorDef);
                        ...}

                private static void registerTransactionManager(Element element, BeanDefinition def) {
                        def.getPropertyValues().add("transactionManagerBeanName",
                        TxNamespaceHandler.getTransactionManagerName(element));
                        }
```

所以这里的事务管理器就是这里配置的transactionManager：

```xml
 <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" mode="proxy"/>

```

最后事务管理器对象存入transactionManagerCache这个缓存集合。

```java
private TransactionManager determineQualifiedTransactionManager(BeanFactory beanFactory, String qualifier) {
		TransactionManager txManager = this.transactionManagerCache.get(qualifier);
		if (txManager == null) {
			txManager = BeanFactoryAnnotationUtils.qualifiedBeanOfType(
					beanFactory, TransactionManager.class, qualifier);
			this.transactionManagerCache.putIfAbsent(qualifier, txManager);
		}
		return txManager;
	}
```

得到事务管理器我们解析阅读创建事务的方法。

## 3.创建事务createTransactionIfNecessary

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
                //通过事务管理器创建事务
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
        //获得事务信息TransactionInfo
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```
我们看下在事务状态TransactionStatus(事务管理器返回的事务对象)上一层的包装对象TransactionInfo(在TransactionInterceptor的内部对象)。
```java

	/**
	 * Opaque object used to hold transaction information. Subclasses
	 * must pass it back to methods on this class, but not see its internals.
	 */
	protected static final class TransactionInfo {
                    //事务管理器
		@Nullable
		private final PlatformTransactionManager transactionManager;
                    //事务属性
		@Nullable
		private final TransactionAttribute transactionAttribute;
                    //切入点
		private final String joinpointIdentification;
                    //事务状态对象，就是事务对象，有事务管理器TransactionManager返回的
		@Nullable
		private TransactionStatus transactionStatus;
                    //TransactionInfo，在TransactionInterceptor的内部对象
		@Nullable
		private TransactionInfo oldTransactionInfo;

		public TransactionInfo(@Nullable PlatformTransactionManager transactionManager,
				@Nullable TransactionAttribute transactionAttribute, String joinpointIdentification) {

			this.transactionManager = transactionManager;
			this.transactionAttribute = transactionAttribute;
			this.joinpointIdentification = joinpointIdentification;
		}

		public PlatformTransactionManager getTransactionManager() {
			Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");
			return this.transactionManager;
		}

		@Nullable
		public TransactionAttribute getTransactionAttribute() {
			return this.transactionAttribute;
		}

		/**
		 * Return a String representation of this joinpoint (usually a Method call)
		 * for use in logging.
		 */
		public String getJoinpointIdentification() {
			return this.joinpointIdentification;
		}

		public void newTransactionStatus(@Nullable TransactionStatus status) {
			this.transactionStatus = status;
		}

		@Nullable
		public TransactionStatus getTransactionStatus() {
			return this.transactionStatus;
		}

		/**
		 * Return whether a transaction was created by this aspect,
		 * or whether we just have a placeholder to keep ThreadLocal stack integrity.
		 */
		public boolean hasTransaction() {
			return (this.transactionStatus != null);
		}
            //将事务信息TransactionInfo绑定到执行线程中
		private void bindToThread() {
			// Expose current TransactionStatus, preserving any existing TransactionStatus
			// for restoration after this transaction is complete.
			this.oldTransactionInfo = transactionInfoHolder.get();
			transactionInfoHolder.set(this);
		}
            //绑定到执行线程中的事务信息恢复到旧值
		private void restoreThreadLocalStatus() {
			// Use stack to restore old transaction TransactionInfo.
			// Will be null if none was set.
			transactionInfoHolder.set(this.oldTransactionInfo);
		}

		
	}

```
可以看到这里获得事务管理器创建的事务对象TransactionStatus之后，再次封装为事务对象TransactionInfo，并存入一个TheadLocal变量绑定到当前线程中。
我们看下新建的事务信息。


![txInfo](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-19-spring_source_code_reading_18/txInfo.png)

获得事务后，开始执行目标方法或者拦截器链的方法“retVal = invocation.proceedWithInvocation()”。得到执行结果retVal。<br>
若抛出异常则进入completeTransactionAfterThrowing方法，正常返回则执行commitTransactionAfterReturning。<br>
我们开始阅读completeTransactionAfterThrowing方法。<br>

## 3.异常处理completeTransactionAfterThrowing
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
                    //回滚
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
			}
			else {
				// We don't roll back on this exception.
				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
				try {
                    //提交
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
			}
		}
	}
```
可以看到，这里支持回滚的异常对象才进行回滚操作，回滚也是通过事务管理器来执行的。<br>
默认支持回滚的异常是RuntimeException和Error。
```java
	@Override
	public boolean rollbackOn(Throwable ex) {
		return (ex instanceof RuntimeException || ex instanceof Error);
	}
```

我们再进入正常结果处理的commitTransactionAfterReturning方法。

## 4.正常结果处理commitTransactionAfterReturning

 这里就很简单了，直接使用事务管理器的提交操作。
```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
			}
			txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
		}
	}

```

这里简单总结一下，事务拦截器主要是拦截目标方法，执行增强处理逻辑，添加增强处理逻辑之后的事务处理逻辑主要是：<br>
1.获得事务属性。<br>
2.获得事务管理器。<br>
3.创建/加入事务。<br>
4.执行目标方法或者拦截器链路的下一个拦截器。<br>
5.异常处理：回滚或者提交。<br>
6.正常结果处理：提交事务。<br>

本篇文章中经常提及到的事务管理器TransactionManager是托管事务对象的核心类，提供新建事务、提交事务、回滚事务、唤醒事务、挂起事务等核心的事务方
法，TransactionStatus就是事务对象，TransactionDefinition是事务定义信息/事务属性，TransactionAttribute事务属性是
TransactionDefinition的子类。

# 二、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)<br>