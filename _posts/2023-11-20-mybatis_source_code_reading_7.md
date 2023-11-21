---
layout: post
title: "Mybatis源码解读七:数据源、连接池和事务"
date: 2023-11-20
tags: [ mybatis ]
comments: true
author: zouhuanli
---

本文是mybatis源码阅读计划的第六文章,本文简单解读Mybatis的数据源、连接池和事务等功能。

本文源码地址为:[https://github.com/zouhuanli/MySpringBoot.git](https://github.com/zouhuanli/MySpringBoot.git)。<br>

严格的讲，数据源、连接池和事务这些不是Mybatis关注的主要的源码。Mybatis是作为一个ORM层来使用的，数据源和连接池都有对应的开源组件，如Druid、Hikari等。<br>
而事务管理器有两种”JDBC|MANAGED“,官网对此的解释是：<br>
JDBC – 这个配置直接使用了 JDBC 的提交和回滚功能，它依赖从数据源获得的连接来管理事务作用域。<br>
MANAGED – 这个配置几乎没做什么。它从不提交或回滚一个连接，而是让容器来管理事务的整个生命周期。<br>

在和Spring结合使用时候没有必要配置事务管理器，将事务管理托管给Spring即可。要注意一个数据源要配置一个SqlSessionFactory和TransactionManager,默认的事务管理是不能跨数据源使用的。

下面按照SpringBoot的默认的HikariDataSource，和Spring的TransactionManager来解读一下Mybatis事务相关的源码

## 一、数据源和连接池

DataSource源码如下：
```java
public interface DataSource  extends CommonDataSource, Wrapper {

  /**
   * <p>Attempts to establish a connection with the data source that
   * this {@code DataSource} object represents.
   *
   * @return  a connection to the data source
   * @throws SQLException if a database access error occurs
   * @throws java.sql.SQLTimeoutException  when the driver has determined that the
   * timeout value specified by the {@code setLoginTimeout} method
   * has been exceeded and has at least tried to cancel the
   * current database connection attempt
   */
  Connection getConnection() throws SQLException;

  /**
   * <p>Attempts to establish a connection with the data source that
   * this {@code DataSource} object represents.
   *
   * @param username the database user on whose behalf the connection is
   *  being made
   * @param password the user's password
   * @return  a connection to the data source
   * @throws SQLException if a database access error occurs
   * @throws java.sql.SQLTimeoutException  when the driver has determined that the
   * timeout value specified by the {@code setLoginTimeout} method
   * has been exceeded and has at least tried to cancel the
   * current database connection attempt
   * @since 1.4
   */
  Connection getConnection(String username, String password)
    throws SQLException;

  /**
   * {@inheritDoc}
   * @since 1.4
   */
  @Override
  java.io.PrintWriter getLogWriter() throws SQLException;

  /**
   * {@inheritDoc}
   * @since 1.4
   */
  @Override
  void setLogWriter(java.io.PrintWriter out) throws SQLException;

  /**
   * {@inheritDoc}
   * @since 1.4
   */
  @Override
  void setLoginTimeout(int seconds) throws SQLException;

  /**
   * {@inheritDoc}
   * @since 1.4
   */
  @Override
  int getLoginTimeout() throws SQLException;

  // JDBC 4.3

  /**
   * Create a new {@code ConnectionBuilder} instance
   * @implSpec
   * The default implementation will throw a {@code SQLFeatureNotSupportedException}
   * @return The ConnectionBuilder instance that was created
   * @throws SQLException if an error occurs creating the builder
   * @throws SQLFeatureNotSupportedException if the driver does not support sharding
   * @since 9
   * @see ConnectionBuilder
   */
  default ConnectionBuilder createConnectionBuilder() throws SQLException {
        throw new SQLFeatureNotSupportedException("createConnectionBuilder not implemented");
  };

}

```
这个顶层接口主要提供获得连接、超时时间等功能，其他如CommonDataSource主要提供日志相关功能。而Wrapper修饰器，提供修饰和解修饰的功能。

HikariDataSource类继承关系如下：
![HikariDataSource](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-11-20-mybatis_source_code_reading_7/HikariDataSource.png)

HikariDataSource继承自DataSource，HikariConfig主要提供相关配置。
先看下最重要的getConnection方法：

```java
@Override
   public Connection getConnection() throws SQLException
   {
      if (isClosed()) {
         throw new SQLException("HikariDataSource " + this + " has been closed.");
      }

      if (fastPathPool != null) {
         return fastPathPool.getConnection();
      }

      // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
      HikariPool result = pool;
      if (result == null) {
         synchronized (this) {
            result = pool;
            if (result == null) {
               validate();
               LOGGER.info("{} - Starting...", getPoolName());
               try {
                                    //创建连接池
                  pool = result = new HikariPool(this);
                  this.seal();
               }
               catch (PoolInitializationException pie) {
                  if (pie.getCause() instanceof SQLException) {
                     throw (SQLException) pie.getCause();
                  }
                  else {
                     throw pie;
                  }
               }
               LOGGER.info("{} - Start completed.", getPoolName());
            }
         }
      }
                                //获取连接
      return result.getConnection();
   }
```

继续进入HikariPool#getConnection方法：

```java
/**
    * Get a connection from the pool, or timeout after the specified number of milliseconds.
    *
    * @param hardTimeout the maximum time to wait for a connection from the pool
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
   public Connection getConnection(final long hardTimeout) throws SQLException
   {
      suspendResumeLock.acquire();
      final var startTime = currentTime();

      try {
         var timeout = hardTimeout;
         do {
            var poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
            if (poolEntry == null) {
               break; // We timed out... break and throw exception
            }

            final var now = currentTime();
            if (poolEntry.isMarkedEvicted() || (elapsedMillis(poolEntry.lastAccessed, now) > aliveBypassWindowMs && isConnectionDead(poolEntry.connection))) {
               closeConnection(poolEntry, poolEntry.isMarkedEvicted() ? EVICTED_CONNECTION_MESSAGE : DEAD_CONNECTION_MESSAGE);
               timeout = hardTimeout - elapsedMillis(startTime);
            }
            else {
               metricsTracker.recordBorrowStats(poolEntry, startTime);
                            //这里创建Connection的代理对象ProxyConnection
               return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry));
            }
         } while (timeout > 0L);

         metricsTracker.recordBorrowTimeoutStats(startTime);
         throw createTimeoutException(startTime);
      }
      catch (InterruptedException e) {
         Thread.currentThread().interrupt();
         throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
      }
      finally {
         suspendResumeLock.release();
      }
   }
```

可以看到最终返回的是ProxyConnection这个代理对象。

而其close方法如下：
```java
  @Override
   public final void close() throws SQLException
   {
      // Closing statements can cause connection eviction, so this must run before the conditional below
      closeStatements();

      if (delegate != ClosedConnection.CLOSED_CONNECTION) {
         leakTask.cancel();

         try {
            if (isCommitStateDirty && !isAutoCommit) {
               delegate.rollback();
               LOGGER.debug("{} - Executed rollback on connection {} due to dirty commit state on close().", poolEntry.getPoolName(), delegate);
            }

            if (dirtyBits != 0) {
               poolEntry.resetConnectionState(this, dirtyBits);
            }

            delegate.clearWarnings();
         }
         catch (SQLException e) {
            // when connections are aborted, exceptions are often thrown that should not reach the application
            if (!poolEntry.isMarkedEvicted()) {
               throw checkException(e);
            }
         }
         finally {
            delegate = ClosedConnection.CLOSED_CONNECTION;
                                 //归还连接到连接池
            poolEntry.recycle();
         }
      }
   }
```

而MybatisAutoConfiguration在创建SqlSessionFactory的DataSource，并不是必须要是HikariDataSource。创建的具体DataSource类型在DataSourceAutoConfiguration指定即可。


## 二、事务

Mybatis和Spring结合使用时，其事务功能托管到Spring管理，其源码是SpringManagedTransaction。其源码如下：
```java
public class SpringManagedTransaction implements Transaction {

  private static final Logger LOGGER = LoggerFactory.getLogger(SpringManagedTransaction.class);

  private final DataSource dataSource;

  private Connection connection;

  private boolean isConnectionTransactional;

  private boolean autoCommit;

  public SpringManagedTransaction(DataSource dataSource) {
    notNull(dataSource, "No DataSource specified");
    this.dataSource = dataSource;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Connection getConnection() throws SQLException {
    if (this.connection == null) {
      openConnection();
    }
    return this.connection;
  }

  /**
   * Gets a connection from Spring transaction manager and discovers if this {@code Transaction} should manage
   * connection or let it to Spring.
   * <p>
   * It also reads autocommit setting because when using Spring Transaction MyBatis thinks that autocommit is always
   * false and will always call commit/rollback so we need to no-op that calls.
   */
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

    LOGGER.debug(() -> "JDBC Connection [" + this.connection + "] will"
        + (this.isConnectionTransactional ? " " : " not ") + "be managed by Spring");
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void commit() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
      LOGGER.debug(() -> "Committing JDBC Connection [" + this.connection + "]");
      this.connection.commit();
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void rollback() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
      LOGGER.debug(() -> "Rolling back JDBC Connection [" + this.connection + "]");
      this.connection.rollback();
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void close() throws SQLException {
    DataSourceUtils.releaseConnection(this.connection, this.dataSource);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Integer getTimeout() throws SQLException {
    ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (holder != null && holder.hasTimeout()) {
      return holder.getTimeToLiveInSeconds();
    }
    return null;
  }

}
```
可以看到这里的connection是由spring-jdbc的DataSourceUtils获取的，然后由其提交事务、回滚事务、关闭连接。

上面已经解读了示例过程的connection是Hikari的ProxyConnection，关闭连接会归还连接到HikariPool。


### 2.1 获得连接

来到SqlSessionTemplate$SqlSessionInterceptor这个类,其getSqlSession如下：

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
                            //已存在的SqlSession，直接返回
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }
                            //创建一个新的SqlSession
    LOGGER.debug(() -> "Creating a new SqlSession");
    session = sessionFactory.openSession(executorType);
                            //注册SqlSession到Spring管理的同步信息管理器
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

跟踪openSession来到DefaultSqlSessionFactory，新创建SqlSession方法如下：

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
      boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

这里会创建SpringManagedTransaction这个事务对象。

而注册事务同步信息到TransactionSynchronizationManager源码如下：

```java
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);
        TransactionSynchronizationManager
            .registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
```

然后跟踪SQL方法的执行流程来到SimpleExecutor；
```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
```

这里getConnection获得连接由SpringManagedTransaction去获取，如上面SpringManagedTransaction的源码，是转发到Spring去处理。

```java
protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    }
    return connection;
  }
```

### 2.2 开启事务

Spring开启事务的源码是AbstractPlatformTransactionManager#getTransaction：

```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException {

		// Use defaults if no transaction definition given.
		TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

		Object transaction = doGetTransaction();
		boolean debugEnabled = logger.isDebugEnabled();
                                    //已有事务的处理
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(def, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
			}
			try {
                                //新开启一个事务
				return startTransaction(def, transaction, debugEnabled, suspendedResources);
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + def);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
		}
	}

```
这里参考阅读一下笔者的这篇文章[Spring源码阅读十九:TransactionManager事务管理器](https://zouhuanli.github.io/spring_source_code_reading_19/)。

### 2.3 提交事务

DefaultSqlSession#commit如下：

```java
@Override
  public void commit(boolean force) {
    try {
      executor.commit(isCommitOrRollbackRequired(force));
      dirty = false;
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

再来到BaseExecutor#commit方法：

```java
@Override
public void commit(boolean required) throws SQLException {
        if (closed) {
                throw new ExecutorException("Cannot commit, transaction is already closed");
        }
                        //清理本地缓存（一级缓存）
            clearLocalCache();
                        //刷新Statement
            flushStatements();
        if (required) {
                        //事务提交
             transaction.commit();
            }
        }
```
最后也是使用connection的commit方法。

### 2.4 回滚事务

再来到BaseExecutor#rollback方法：

```java
@Override
  public void rollback(boolean required) throws SQLException {
    if (!closed) {
      try {
        clearLocalCache();
        flushStatements(true);
      } finally {
        if (required) {
          transaction.rollback();
        }
      }
    }
  }
```

SpringManagedTransaction#rollback方法如下：

```java
/**
 * {@inheritDoc}
 */
@Override
public void rollback() throws SQLException {
        if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
        LOGGER.debug(() -> "Rolling back JDBC Connection [" + this.connection + "]");
        this.connection.rollback();
        }
        }
```

### 2.5 关闭连接

关闭连接的方法如下：

```java
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    if ((holder != null) && (holder.getSqlSession() == session)) {
                        //释放连接
      LOGGER.debug(() -> "Releasing transactional SqlSession [" + session + "]");
      holder.released();
    } else {
      LOGGER.debug(() -> "Closing non transactional SqlSession [" + session + "]");
                        //关闭连接
      session.close();
    }
  }
```

Executor和SpringManagedTransaction的关闭方法如下：

```java

  @Override
  public void close(boolean forceRollback) {
    try {
      try {
        rollback(forceRollback);
      } finally {
        if (transaction != null) {
          transaction.close();
        }
      }
    } catch (SQLException e) {
      // Ignore. There's nothing that can be done at this point.
      log.warn("Unexpected exception on closing transaction.  Cause: " + e);
    } finally {
                    //释放资源
      transaction = null;
      deferredLoads = null;
      localCache = null;
      localOutputParameterCache = null;
      closed = true;
    }
  }
```

```java
@Override
  public void close() throws SQLException {
    DataSourceUtils.releaseConnection(this.connection, this.dataSource);
  }
```

### 2.6 简单总结

从上面源码可以看出，Mybatis和spring结合使用时候，mybatis的事务功能全部托管给Spring去管理，自己基本啥都没做，主要是做一些资源绑定和资源清理。

## 三、参考材料

1. https://mybatis.org/mybatis-3/zh/ <br>
2. Mybatis源码(版本3.5.13) <br>

