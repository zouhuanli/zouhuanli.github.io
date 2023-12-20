---
layout: post
title: "Tomcat源码解读四:Server和Service"
date: 2023-12-04
tags: [ Tomcat ]
comments: true
author: zouhuanli
---

本文简单解读Tomcat的Server和Service组件。Server服务器，一个Tomcat应用只会创建和启动一个Server；Service服务,一个Server可以创建和启动多个Service。
本系列的源码工程为：[HowTomcatWorksSourceCode](https://github.com/zouhuanli/HowTomcatWorksSourceCode.git)。

# 一、Server

Server服务器组件，包含了其他的所有组件，提供方法来启动和关闭整个Tomcat系统。
我们看下StandardServer的主要方法：
````java

/**
 * Standard implementation of the <b>Server</b> interface, available for use (but not required) when deploying and
 * starting Catalina.
 *
 * @author Craig R. McClanahan
 */
public final class StandardServer extends LifecycleMBeanBase implements Server {


    /**
     * The port number on which we wait for shutdown commands.
     */
    private int port = 8005;


    /**
     * The set of Services associated with this Server.
     */
                     //Service列表
    private Service[] services = new Service[0];
    private final Object servicesLock = new Object();


    /**
     * The shutdown command string we are looking for.
     */
                     //关机命令
    private String shutdown = "SHUTDOWN";


    public void stopAwait() {
        stopAwait = true;
        Thread t = awaitThread;
        if (t != null) {
            ServerSocket s = awaitSocket;
            if (s != null) {
                awaitSocket = null;
                try {
                    s.close();
                } catch (IOException e) {
                    // Ignored
                }
            }
            t.interrupt();
            try {
                t.join(1000);
            } catch (InterruptedException e) {
                // Ignored
            }
        }
    }

    /**
     * Wait until a proper shutdown command is received, then return. This keeps the main thread alive - the thread pool
     * listening for http connections is daemon threads.
     */
                            //等待关闭整个Tomcat
    @Override
    public void await() {
        // Negative values - don't wait on port - tomcat is embedded or we just don't like ports
        if (getPortWithOffset() == -2) {
            // undocumented yet - for embedding apps that are around, alive.
            return;
        }
        Thread currentThread = Thread.currentThread();
        if (getPortWithOffset() == -1) {
            try {
                awaitThread = currentThread;
                while (!stopAwait) {
                    try {
                        Thread.sleep(10000);
                    } catch (InterruptedException ex) {
                        // continue and check the flag
                    }
                }
            } finally {
                awaitThread = null;
            }
            return;
        }

        // Set up a server socket to wait on
        try {
            awaitSocket = new ServerSocket(getPortWithOffset(), 1, InetAddress.getByName(address));
        } catch (IOException e) {
            log.error(sm.getString("standardServer.awaitSocket.fail", address, String.valueOf(getPortWithOffset()),
                    String.valueOf(getPort()), String.valueOf(getPortOffset())), e);
            return;
        }

        try {
            awaitThread = currentThread;

            // Loop waiting for a connection and a valid command
            while (!stopAwait) {
                ServerSocket serverSocket = awaitSocket;
                if (serverSocket == null) {
                    break;
                }

                // Wait for the next connection
                Socket socket = null;
                StringBuilder command = new StringBuilder();
                try {
                    InputStream stream;
                    long acceptStartTime = System.currentTimeMillis();
                    try {
                        socket = serverSocket.accept();
                        socket.setSoTimeout(10 * 1000); // Ten seconds
                        stream = socket.getInputStream();
                    } catch (SocketTimeoutException ste) {
                        // This should never happen but bug 56684 suggests that
                        // it does.
                        log.warn(sm.getString("standardServer.accept.timeout",
                                Long.valueOf(System.currentTimeMillis() - acceptStartTime)), ste);
                        continue;
                    } catch (IOException e) {
                        if (stopAwait) {
                            // Wait was aborted with socket.close()
                            break;
                        }
                        log.error(sm.getString("standardServer.accept.error"), e);
                        break;
                    }

                    // Read a set of characters from the socket
                    int expected = 1024; // Cut off to avoid DoS attack
                    while (expected < shutdown.length()) {
                        if (random == null) {
                            random = new Random();
                        }
                        expected += random.nextInt() % 1024;
                    }
                    while (expected > 0) {
                        int ch = -1;
                        try {
                            ch = stream.read();
                        } catch (IOException e) {
                            log.warn(sm.getString("standardServer.accept.readError"), e);
                            ch = -1;
                        }
                        // Control character or EOF (-1) terminates loop
                        if (ch < 32 || ch == 127) {
                            break;
                        }
                        command.append((char) ch);
                        expected--;
                    }
                } finally {
                    // Close the socket now that we are done with it
                    try {
                        if (socket != null) {
                            socket.close();
                        }
                    } catch (IOException e) {
                        // Ignore
                    }
                }

                // Match against our command string
                boolean match = command.toString().equals(shutdown);
                if (match) {
                    log.info(sm.getString("standardServer.shutdownViaPort"));
                    break;
                } else {
                    log.warn(sm.getString("standardServer.invalidShutdownCommand", command.toString()));
                }
            }
        } finally {
            ServerSocket serverSocket = awaitSocket;
            awaitThread = null;
            awaitSocket = null;

            // Close the server socket and return
            if (serverSocket != null) {
                try {
                    serverSocket.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
    }


    /**
     * Start nested components ({@link Service}s) and implement the requirements of
     * {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error that prevents this component from being
     *                                   used
     */
                    //start方法，主要是启动service
    @Override
    protected void startInternal() throws LifecycleException {

        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);

        // Initialize utility executor
        synchronized (utilityExecutorLock) {
            reconfigureUtilityExecutor(getUtilityThreadsInternal(utilityThreads));
            register(utilityExecutor, "type=UtilityExecutor");
        }

        globalNamingResources.start();

        // Start our defined Services
        synchronized (servicesLock) {
            for (Service service : services) {
                service.start();
            }
        }

        if (periodicEventDelay > 0) {
            monitorFuture = getUtilityExecutor().scheduleWithFixedDelay(this::startPeriodicLifecycleEvent, 0, 60,
                    TimeUnit.SECONDS);
        }
    }


    /**
     * Stop nested components ({@link Service}s) and implement the requirements of
     * {@link org.apache.catalina.util.LifecycleBase#stopInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error that needs to be reported
     */
                        //关闭方法，主要是关闭service
    @Override
    protected void stopInternal() throws LifecycleException {

        setState(LifecycleState.STOPPING);

        if (monitorFuture != null) {
            monitorFuture.cancel(true);
            monitorFuture = null;
        }
        if (periodicLifecycleEventFuture != null) {
            periodicLifecycleEventFuture.cancel(false);
            periodicLifecycleEventFuture = null;
        }

        fireLifecycleEvent(CONFIGURE_STOP_EVENT, null);

        // Stop our defined Services
        for (Service service : services) {
            service.stop();
        }

        synchronized (utilityExecutorLock) {
            if (utilityExecutor != null) {
                utilityExecutor.shutdownNow();
                unregister("type=UtilityExecutor");
                utilityExecutor = null;
            }
        }

        globalNamingResources.stop();

        stopAwait();
    }

    /**
     * Invoke a pre-startup initialization. This is used to allow connectors to bind to restricted ports under Unix
     * operating environments.
     */
                        //初始化方法，见之前文章
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");

        // Register the MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");

        // Register the naming resources
        globalNamingResources.init();

        // Initialize our defined Services
        for (Service service : services) {
            service.init();
        }
    }

                        //销毁方法，主要是销毁Service
    @Override
    protected void destroyInternal() throws LifecycleException {
        // Destroy our defined Services
        for (Service service : services) {
            service.destroy();
        }

        globalNamingResources.destroy();

        unregister(onameMBeanFactory);

        unregister(onameStringCache);

        super.destroyInternal();
    }


}
````
StandardServer的主要方法都是操作Service，如何关机方法是await里面监听SHUTDOWN命令。

# 二、Service

我们依旧看下StandardService的主要方法。

```java


/**
 * Standard implementation of the <code>Service</code> interface. The associated Container is generally an instance of
 * Engine, but this is not required.
 *
 * @author Craig R. McClanahan
 */

public class StandardService extends LifecycleMBeanBase implements Service {

    /**
     * The <code>Server</code> that owns this Service, if any.
     */
    private Server server = null;

    /**
     * The set of Connectors associated with this Service.
     */
                        //连接器
    protected Connector connectors[] = new Connector[0];
    private final Object connectorsLock = new Object();

    /**
     * The list of executors held by the service.
     */
                        //线程池列表
    protected final ArrayList<Executor> executors = new ArrayList<>();
                        //引擎
    private Engine engine = null;


    /**
     * Start nested components ({@link Executor}s, {@link Connector}s and {@link Container}s) and implement the
     * requirements of {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error that prevents this component from being
     *                                   used
     */
                        //启动方法
    @Override
    protected void startInternal() throws LifecycleException {

        if (log.isInfoEnabled()) {
            log.info(sm.getString("standardService.start.name", this.name));
        }
        setState(LifecycleState.STARTING);

        // Start our defined Container first
        if (engine != null) {
            synchronized (engine) {
                engine.start();
            }
        }

        synchronized (executors) {
            for (Executor executor : executors) {
                executor.start();
            }
        }

        mapperListener.start();

        // Start our defined Connectors second
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            }
        }
    }


    /**
     * Stop nested components ({@link Executor}s, {@link Connector}s and {@link Container}s) and implement the
     * requirements of {@link org.apache.catalina.util.LifecycleBase#stopInternal()}.
     *
     * @exception LifecycleException if this component detects a fatal error that needs to be reported
     */
                        //停止方法
    @Override
    protected void stopInternal() throws LifecycleException {

        synchronized (connectorsLock) {
            // Initiate a graceful stop for each connector
            // This will only work if the bindOnInit==false which is not the
            // default.
            for (Connector connector : connectors) {
                connector.getProtocolHandler().closeServerSocketGraceful();
            }

            // Wait for the graceful shutdown to complete
            long waitMillis = gracefulStopAwaitMillis;
            if (waitMillis > 0) {
                for (Connector connector : connectors) {
                    waitMillis = connector.getProtocolHandler().awaitConnectionsClose(waitMillis);
                }
            }

            // Pause the connectors
            for (Connector connector : connectors) {
                connector.pause();
            }
        }

        if (log.isInfoEnabled()) {
            log.info(sm.getString("standardService.stop.name", this.name));
        }
        setState(LifecycleState.STOPPING);

        // Stop our defined Container once the Connectors are all paused
        if (engine != null) {
            synchronized (engine) {
                engine.stop();
            }
        }

        // Now stop the connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                if (!LifecycleState.STARTED.equals(connector.getState())) {
                    // Connectors only need stopping if they are currently
                    // started. They may have failed to start or may have been
                    // stopped (e.g. via a JMX call)
                    continue;
                }
                connector.stop();
            }
        }

        // If the Server failed to start, the mapperListener won't have been
        // started
        if (mapperListener.getState() != LifecycleState.INITIALIZED) {
            mapperListener.stop();
        }

        synchronized (executors) {
            for (Executor executor : executors) {
                executor.stop();
            }
        }
    }


    /**
     * Invoke a pre-startup initialization. This is used to allow connectors to bind to restricted ports under Unix
     * operating environments.
     */
                        //初始化
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (engine != null) {
            engine.init();
        }

        // Initialize any Executors
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // Initialize mapper listener
        mapperListener.init();

        // Initialize our defined Connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.init();
            }
        }
    }

                        //销毁方法
    @Override
    protected void destroyInternal() throws LifecycleException {
        mapperListener.destroy();

        // Destroy our defined Connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.destroy();
            }
        }

        // Destroy any Executors
        for (Executor executor : findExecutors()) {
            executor.destroy();
        }

        if (engine != null) {
            engine.destroy();
        }

        super.destroyInternal();
    }


}


```

Service中的容器只有一个，connectors连接器可以有多个。"例如一个连接器处理HTTP请求，一个处理HTTPS请求"。

初始化、启动、销毁方法主要是操作Container容器(Engine)、Executor线程池、Connector连接器这三者。

下面对Server和Service做一个简单的测试，来自《深入剖析Tomcat》这本书第14章的例子。

# 三、简单测试

看下原书的源码：

```java
public final class Bootstrap {
  public static void main(String[] args) {

    System.setProperty("catalina.base", System.getProperty("user.dir"));
    Connector connector = new HttpConnector();

    Wrapper wrapper1 = new StandardWrapper();
    wrapper1.setName("Primitive");
    wrapper1.setServletClass("PrimitiveServlet");
    Wrapper wrapper2 = new StandardWrapper();
    wrapper2.setName("Modern");
    wrapper2.setServletClass("ModernServlet");

    Context context = new StandardContext();
    // StandardContext's start method adds a default mapper
    context.setPath("/app1");
    context.setDocBase("app1");

    context.addChild(wrapper1);
    context.addChild(wrapper2);

    LifecycleListener listener = new SimpleContextConfig();
    ((Lifecycle) context).addLifecycleListener(listener);

    Host host = new StandardHost();
    host.addChild(context);
    host.setName("localhost");
    host.setAppBase("webapps");

    Loader loader = new WebappLoader();
    context.setLoader(loader);
    // context.addServletMapping(pattern, name);
    context.addServletMapping("/Primitive", "Primitive");
    context.addServletMapping("/Modern", "Modern");

    Engine engine = new StandardEngine();
    engine.addChild(host);
    engine.setDefaultHost("localhost");

    Service service = new StandardService();
    service.setName("Stand-alone Service");
    Server server = new StandardServer();
    server.addService(service);
    service.addConnector(connector);

    //StandardService class's setContainer will call all its connector's setContainer method
    service.setContainer(engine);

    // Start the new server
    if (server instanceof Lifecycle) {
      try {
        server.initialize();
        ((Lifecycle) server).start();
        server.await();
        // the program waits until the await method returns,
        // i.e. until a shutdown command is received.
      }
      catch (LifecycleException e) {
        e.printStackTrace(System.out);
      }
    }

    // Shut down the server
    if (server instanceof Lifecycle) {
      try {
        ((Lifecycle) server).stop();
      }
      catch (LifecycleException e) {
        e.printStackTrace(System.out);
      }
    }
  }
}
```

创建Server、Service、容器等，初始化完成并启动。然后等待SHUTDOWN关闭命令。

关闭操作如下：
```java
 public static void main(String[] args) {
    // the following code is taken from the Stop method of
    // the org.apache.catalina.startup.Catalina class
    int port = 8005;
    try {
      Socket socket = new Socket("127.0.0.1", port);
      OutputStream stream = socket.getOutputStream();
      String shutdown = "SHUTDOWN";
      for (int i = 0; i < shutdown.length(); i++)
        stream.write(shutdown.charAt(i));
      stream.flush();
      stream.close();
      socket.close();
      System.out.println("The server was successfully shut down.");
    }
    catch (IOException e) {
      System.out.println("Error. The server has not been started.");
    }
  }
```
这里就是向8005端口发送"SHUTDOWN".

然后执行测试,正常关闭Tomcat：

```text
HttpConnector Opening server socket on all host IP addresses
Starting service Stand-alone Service
Apache Tomcat/@VERSION@
WebappLoader[/app1]: Deploying class repositories to work directory D:\git\code\HowTomcatWorksSourceCode\work\_\localhost\app1
StandardManager[/app1]: Seeding random number generator class java.security.SecureRandom
StandardManager[/app1]: Seeding of random number generator has been completed
HttpConnector[8080] Starting background thread
Stopping service Stand-alone Service
HttpConnector[8080] Stopping background thread

Process finished with exit code 0
```

# 四、参考材料

1. 《深入剖析Tomcat》 <br>
2. 《HTTP权威指南》

