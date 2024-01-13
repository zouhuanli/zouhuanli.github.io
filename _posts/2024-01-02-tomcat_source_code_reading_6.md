---
layout: post
title: "Tomcat源码解读六:Context和Wrapper"
date: 2024-01-02
tags: [ Tomcat ]
comments: true
author: zouhuanli
---

本文简单解读Tomcat的Context和Wrapper组件。
本系列的源码工程为：[HowTomcatWorksSourceCode](https://github.com/zouhuanli/HowTomcatWorksSourceCode.git)。

本文目录为：
- [一、概述](#一概述)
- [二、源码解读](#二源码解读)
  - [2，1 StandardContext](#21-standardcontext)
  - [2.2 StandardWrapper](#22-standardwrapper)
- [三、简单测试](#三简单测试)
- [四、参数资料](#四参数资料)

# 一、概述

Context表示一个Web应用,其默认实现类是StandardContext，其对应的阀是StandardContextValve<br>
Wrapper表示一个单独的Servlet，比如普通的经典SpringMVC应用配置的DispatcherServlet和Tomcat自带的DefaultServlet，Wrapper对应的阀则是StandardWrapperValve。

# 二、源码解读

## 2，1 StandardContext

解读StandardContext可以从一个单独的应用的'web.xml'谈起。<br>
比如：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE web-app PUBLIC
        "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd"  >

<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>SpringMVC Demo</display-name>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:springmvc-context.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>


    <servlet>
        <servlet-name>springmvc-demo</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc-demo</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    <filter>
        <filter-name>encodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>encodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>

```
web的xml的配置基本和StandardContext的属性是保持一致的，比如这里是servlet-mapping：
```java

    /**
     * The servlet mappings for this web application, keyed by matching pattern.
     */
    private Map<String,String> servletMappings = new HashMap<>();
```
以及loadOnStartup的操作：
```java
    /**
     * Load and initialize all servlets marked "load on startup" in the web application deployment descriptor.
     *
     * @param children Array of wrappers for all currently defined servlets (including those not declared load on
     *                     startup)
     *
     * @return <code>true</code> if load on startup was considered successful
     */
    public boolean loadOnStartup(Container children[]) {

        // Collect "load on startup" servlets that need to be initialized
        TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap<>();
        for (Container child : children) {
            Wrapper wrapper = (Wrapper) child;
            int loadOnStartup = wrapper.getLoadOnStartup();
            if (loadOnStartup < 0) {
                continue;
            }
            Integer key = Integer.valueOf(loadOnStartup);
            map.computeIfAbsent(key, k -> new ArrayList<>()).add(wrapper);
        }
        //.......
    }
```
总结的说就是解析web.xml来生成Context的各项属性。

要了解的是，一个Context是可以有多个Servlet的。<br>
下面再来解读一些StandardContext的源码,这里init、start、stop不作详细解读了。
在Context的启动方法中启动了很多组件，这里列举一个监听器的启动代码：
```java
 public boolean listenerStart() {

       
        // Instantiate the required listeners
        String listeners[] = findApplicationListeners();
        Object results[] = new Object[listeners.length];
        boolean ok = true;
        for (int i = 0; i < results.length; i++) {
            if (getLogger().isDebugEnabled()) {
                getLogger().debug(" Configuring event listener class '" + listeners[i] + "'");
            }
            try {
                String listener = listeners[i];
                //创建监听器
                results[i] = getInstanceManager().newInstance(listener);
            } catch (Throwable t) {
                t = ExceptionUtils.unwrapInvocationTargetException(t);
                ExceptionUtils.handleThrowable(t);
                getLogger().error(sm.getString("standardContext.applicationListener", listeners[i]), t);
                ok = false;
            }
        }
        if (!ok) {
            getLogger().error(sm.getString("standardContext.applicationSkipped"));
            return false;
        }

        List<Object> eventListeners = new ArrayList<>();
        List<Object> lifecycleListeners = new ArrayList<>();
        for (Object result : results) {
            if ((result instanceof ServletContextAttributeListener) ||
                    (result instanceof ServletRequestAttributeListener) || (result instanceof ServletRequestListener) ||
                    (result instanceof HttpSessionIdListener) || (result instanceof HttpSessionAttributeListener)) {
                eventListeners.add(result);
            }
            if ((result instanceof ServletContextListener) || (result instanceof HttpSessionListener)) {
                lifecycleListeners.add(result);
            }
        }

        eventListeners.addAll(Arrays.asList(getApplicationEventListeners()));
        setApplicationEventListeners(eventListeners.toArray());
        for (Object lifecycleListener : getApplicationLifecycleListeners()) {
            lifecycleListeners.add(lifecycleListener);
            if (lifecycleListener instanceof ServletContextListener) {
                noPluggabilityListeners.add(lifecycleListener);
            }
        }
        setApplicationLifecycleListeners(lifecycleListeners.toArray());

        // 获得ServletContext
        getServletContext();
        context.setNewServletContextListenerAllowed(false);

        Object instances[] = getApplicationLifecycleListeners();
        if (instances == null || instances.length == 0) {
            return ok;
        }

        ServletContextEvent event = new ServletContextEvent(getServletContext());
        ServletContextEvent tldEvent = null;
        if (noPluggabilityListeners.size() > 0) {
            noPluggabilityServletContext = new NoPluggabilityServletContext(getServletContext());
            tldEvent = new ServletContextEvent(noPluggabilityServletContext);
        }
        for (Object instance : instances) {
            if (!(instance instanceof ServletContextListener)) {
                continue;
            }
            ServletContextListener listener = (ServletContextListener) instance;
            try {
                fireContainerEvent("beforeContextInitialized", listener);
                    //监听器处理对应的事件
                if (noPluggabilityListeners.contains(listener)) {
                    listener.contextInitialized(tldEvent);
                } else {
                    listener.contextInitialized(event);
                }
                fireContainerEvent("afterContextInitialized", listener);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                fireContainerEvent("afterContextInitialized", listener);
                getLogger().error(sm.getString("standardContext.listenerStart", instance.getClass().getName()), t);
                ok = false;
            }
        }
        return ok;

    }

```
上面代码创建了监听器并执行了对应的监听事件。

然后我们关注一下Context的子容器Wrapper(Servlet)相关的源码：
```java
 @Override
    public void addChild(Container child) {
        if (log.isDebugEnabled()) {
            log.debug("Add child " + child + " " + this);
        }
                            //添加子容器Wrapper(Servlet)
        synchronized (children) {
            if (children.get(child.getName()) != null) {
                throw new IllegalArgumentException(sm.getString("containerBase.child.notUnique", child.getName()));
            }
            child.setParent(this); // May throw IAE
            children.put(child.getName(), child);
        }

        fireContainerEvent(ADD_CHILD_EVENT, child);

        // Start child
        // Don't do this inside sync block - start can be a slow process and
        // locking the children object can cause problems elsewhere
        try {
                        //子容器Wrapper(Servlet)启动
            if ((getState().isAvailable() || LifecycleState.STARTING_PREP.equals(getState())) && startChildren) {
                child.start();
            }
        } catch (LifecycleException e) {
            throw new IllegalStateException(sm.getString("containerBase.child.start"), e);
        }
    }

    public boolean loadOnStartup(Container children[]) {

    // Collect "load on startup" servlets that need to be initialized
    TreeMap<Integer,ArrayList<Wrapper>> map = new TreeMap<>();
    for (Container child : children) {
        Wrapper wrapper = (Wrapper) child;
        int loadOnStartup = wrapper.getLoadOnStartup();
        if (loadOnStartup < 0) {
            continue;
        }
        Integer key = Integer.valueOf(loadOnStartup);
        map.computeIfAbsent(key, k -> new ArrayList<>()).add(wrapper);
    }

    // Load the collected "load on startup" servlets
    for (ArrayList<Wrapper> list : map.values()) {
        for (Wrapper wrapper : list) {
            try {
                        //执行子容器Wrapper(Servlet)的load操作
                wrapper.load();
            } catch (ServletException e) {
                getLogger().error(
                        sm.getString("standardContext.loadOnStartup.loadException", getName(), wrapper.getName()),
                        StandardWrapper.getRootCause(e));
                // NOTE: load errors (including a servlet that throws
                // UnavailableException from the init() method) are NOT
                // fatal to application startup
                // unless failCtxIfServletStartFails="true" is specified
                if (getComputedFailCtxIfServletStartFails()) {
                    return false;
                }
            }
        }
    }
    return true;

}
```
上面的源码创建Wrapper，并进行子容器Wrapper(Servlet)的load操作。<br>
我们继续来到StandardWrapper的源码。

## 2.2 StandardWrapper

<strong>Wrapper表示一个独立的Servlet</strong>,StandardWrapper是其默认实现类。<br>

StandardWrapper最主要的功能就载入Servlet类，实例化并执行初始化(init)方法。但实际处理一个Http请求是由StandardWrapperValve的invoke处理的，而不是StandardWrapper。<br>
StandardWrapperValve通过StandardWrapper的allocate方法获得构造好的Servlet实例，其源码如下：
```java

    /**
     * Allocate an initialized instance of this Servlet that is ready to have its <code>service()</code> method called.
     *
     * @exception ServletException if the servlet init() method threw an exception
     * @exception ServletException if a loading error occurs
     */
    @Override
    public Servlet allocate() throws ServletException {

        // If we are currently unloading this servlet, throw an exception
        if (unloading) {
            throw new ServletException(sm.getString("standardWrapper.unloading", getName()));
        }

        boolean newInstance = false;

        // Load and initialize our instance if necessary
        if (instance == null || !instanceInitialized) {
            synchronized (this) {
                if (instance == null) {
                    try {
                        if (log.isDebugEnabled()) {
                            log.debug("Allocating instance");
                        }
                        instance = loadServlet();
                        newInstance = true;
                        // Increment here to prevent a race condition
                        // with unload. Bug 43683, test case #3
                        countAllocated.incrementAndGet();
                    } catch (ServletException e) {
                        throw e;
                    } catch (Throwable e) {
                        ExceptionUtils.handleThrowable(e);
                        throw new ServletException(sm.getString("standardWrapper.allocate"), e);
                    }
                }
                if (!instanceInitialized) {
                    initServlet(instance);
                }
            }
        }

        if (log.isTraceEnabled()) {
            log.trace("  Returning instance");
        }
        // For new instances, count will have been incremented at the
        // time of creation
        if (!newInstance) {
            countAllocated.incrementAndGet();
        }
        return instance;
    }
```
加载(实例化)和初始化(init)Servlet的源码如下:
```java
/**
     * Load and initialize an instance of this servlet, if there is not already an initialized instance. This can be
     * used, for example, to load servlets that are marked in the deployment descriptor to be loaded at server startup
     * time.
     *
     * @return the loaded Servlet instance
     *
     * @throws ServletException for a Servlet load error
     */
    public synchronized Servlet loadServlet() throws ServletException {

        // Nothing to do if we already have an instance or an instance pool
        if (instance != null) {
            return instance;
        }

        PrintStream out = System.out;
        if (swallowOutput) {
            SystemLogHandler.startCapture();
        }

        Servlet servlet;
        try {
            long t1 = System.currentTimeMillis();
            // Complain if no servlet class has been specified
            if (servletClass == null) {
                unavailable(null);
                throw new ServletException(sm.getString("standardWrapper.notClass", getName()));
            }

            InstanceManager instanceManager = ((StandardContext) getParent()).getInstanceManager();
            try {
                servlet = (Servlet) instanceManager.newInstance(servletClass);
            } catch (ClassCastException e) {
                unavailable(null);
                // Restore the context ClassLoader
                throw new ServletException(sm.getString("standardWrapper.notServlet", servletClass), e);
            } catch (Throwable e) {
                e = ExceptionUtils.unwrapInvocationTargetException(e);
                ExceptionUtils.handleThrowable(e);
                unavailable(null);

                // Added extra log statement for Bugzilla 36630:
                // https://bz.apache.org/bugzilla/show_bug.cgi?id=36630
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("standardWrapper.instantiate", servletClass), e);
                }

                // Restore the context ClassLoader
                throw new ServletException(sm.getString("standardWrapper.instantiate", servletClass), e);
            }

            if (multipartConfigElement == null) {
                MultipartConfig annotation = servlet.getClass().getAnnotation(MultipartConfig.class);
                if (annotation != null) {
                    multipartConfigElement = new MultipartConfigElement(annotation);
                }
            }

            // Special handling for ContainerServlet instances
            // Note: The InstanceManager checks if the application is permitted
            // to load ContainerServlets
            if (servlet instanceof ContainerServlet) {
                ((ContainerServlet) servlet).setWrapper(this);
            }

            classLoadTime = (int) (System.currentTimeMillis() - t1);

            initServlet(servlet);

            fireContainerEvent("load", this);

            loadTime = System.currentTimeMillis() - t1;
        } finally {
            if (swallowOutput) {
                String log = SystemLogHandler.stopCapture();
                if (log != null && log.length() > 0) {
                    if (getServletContext() != null) {
                        getServletContext().log(log);
                    } else {
                        out.println(log);
                    }
                }
            }
        }
        return servlet;

    }

    private synchronized void initServlet(Servlet servlet) throws ServletException {

        if (instanceInitialized) {
            return;
        }

        // Call the initialization method of this servlet
        try {
            servlet.init(facade);
            instanceInitialized = true;
        } catch (UnavailableException f) {
            unavailable(f);
            throw f;
        } catch (ServletException f) {
            // If the servlet wanted to be unavailable it would have
            // said so, so do not call unavailable(null).
            throw f;
        } catch (Throwable f) {
            ExceptionUtils.handleThrowable(f);
            getServletContext().log(sm.getString("standardWrapper.initException", getName()), f);
            // If the servlet wanted to be unavailable it would have
            // said so, so do not call unavailable(null).
            throw new ServletException(sm.getString("standardWrapper.initException", getName()), f);
        }
    }
```
上面代码主要就是使用反射创建Servlet然后执行init方法。<br>
我们回到DispatcherServlet的顶层接口Servlet的定义：
```java
public interface Servlet {

    public void init(ServletConfig config) throws ServletException;

 
    public ServletConfig getServletConfig();

    //......

}
```
从这里联系我们知道了Servlet的创建和初始化是由Servlet容器来执行的，以及ServletConfig也是容器创建的。而这里的ServletConfig类的对象facade最终指向了ApplicationContextFacade。
ApplicationContextFacade也实现了ServletContext。

# 三、简单测试

这里以原书第11章的源码坐下简单的测试：
```java
public final class Bootstrap {
  public static void main(String[] args) {

  //invoke: http://localhost:8080/Modern or  http://localhost:8080/Primitive

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
    context.setPath("/myApp");
    context.setDocBase("myApp");
    LifecycleListener listener = new SimpleContextConfig();
    ((Lifecycle) context).addLifecycleListener(listener);

    context.addChild(wrapper1);
    context.addChild(wrapper2);
    // for simplicity, we don't add a valve, but you can add
    // valves to context or wrapper just as you did in Chapter 6

    Loader loader = new WebappLoader();
    context.setLoader(loader);
    // context.addServletMapping(pattern, name);
    context.addServletMapping("/Primitive", "Primitive");
    context.addServletMapping("/Modern", "Modern");
    // add ContextConfig. This listener is important because it configures
    // StandardContext (sets configured to true), otherwise StandardContext
    // won't start
    connector.setContainer(context);
    try {
      connector.initialize();
      ((Lifecycle) connector).start();
      ((Lifecycle) context).start();

      // make the application wait until we press a key.
      System.in.read();
      ((Lifecycle) context).stop();
    }
    catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```
上面的测试代码中Context内设置了两个Wrapper，输入http://localhost:8080/Primitive执行测试。<br>
正常成功返回:
```text
HTTP/1.1 200 ok 
Content-Type: text/html

<html><h1></h1><body><p>Hello. Roses are red.</p></body></html>
```

# 四、参数资料

1.《深入剖析Tomcat》

