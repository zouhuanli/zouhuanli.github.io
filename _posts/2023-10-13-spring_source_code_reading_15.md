---
layout: post
title: "Spring源码阅读十五:DispatcherServlet的请求处理流程"
date: 2023-10-13
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十五篇文章，本文着重介绍DispatcherServlet的请求处理流程,也就是DispatcherServlet.doDispatch()
方法。<br>
上一篇文章已经介绍了SpringMVC的简单使用和DispatcherServlet的初始化流程，本文开始阅读doDispatch()方法的方法。
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>
我们从Servlet的service方法开始阅读。

# 一、SpringMVC的请求流程

重新简单介绍一下SpringMVC的请求流程，也就是DispatcherServlet的doDispatch方法。<br>
还是开涛老师的图，如下：

![spring-springframework-mvc](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-12-spring_source_code_reading_14/spring-springframework-mvc.png)

主要流程如下：

1.用户(前端)请求DispatcherServlet，DispatcherServlet作为核心的前端控制器，自己不处理请求，只做主流程串联，进行请求分派。<br>
2.DispatcherServlet将请求转发给HandlerMapping，handlerMapping返回HandlerExecutionChain（包含Handler控制器和拦截器列表）。<br>
3.DispatcherServlet再将Handler控制器转发给HandlerAdapter，获得Handler的适配器。<br>
4.HandlerAdapter真正请求处理器的方法，返回ModelAndView对象。也能不返回ModelAndView对象，直接写回response数据，<br>如@ResponseBody，RequestMappingHandlerAdapter调用Handler的目标方法之后，直接返回null，后面的和View对象的流程就不用了。<br>
5.ModelAndView再进行视图解析，解析为具体的视图View。<br>
6.视图View执行render方法进行渲染，Model对象是一个Map结构。<br>
7.DispatcherServlet将渲染后的数据返回给用户(前端)。<br>

笔者现结合DispatcherServlet源码详细介绍请求流程。先从Servlet的service()方法作为入口开始阅读。


# 二、DispatcherServlet的请求处理流程
## 1.service()方法

service方法如下

```java
 /**
     * Called by the servlet container to allow the servlet to respond to a request.
     *
     * <p>
     * This method is only called after the servlet's <code>init()</code> method has completed successfully.
     * 
     * <p>
     * The status code of the response always should be set for a servlet that throws or sends an error.
     *
     * 
     * <p>
     * Servlets typically run inside multithreaded servlet containers that can handle multiple requests concurrently.
     * Developers must be aware to synchronize access to any shared resources such as files, network connections, and as
     * well as the servlet's class and instance variables.
     *
     * @param req the <code>ServletRequest</code> object that contains the client's request
     *
     * @param res the <code>ServletResponse</code> object that contains the servlet's response
     *
     * @exception ServletException if an exception occurs that interferes with the servlet's normal operation
     *
     * @exception IOException if an input or output exception occurs
     *
     */
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;
```
这里的ServletRequest、ServletResponse不仅仅是http协议的请求，可以是任何的协议的Web请求。<br>
GenericServlet的service()方法是空方法，继续阅读子类。<br>
HttpServletRequest的service()方法如下，可以看到主要就是判断是否Http协议的请求。
```java
/**
     * Dispatches client requests to the protected <code>service</code> method. There's no need to override this method.
     * 
     * @param req the {@link HttpServletRequest} object that contains the request the client made of the servlet
     *
     * @param res the {@link HttpServletResponse} object that contains the response the servlet returns to the client
     *
     * @throws IOException if an input or output error occurs while the servlet is handling the HTTP request
     *
     * @throws ServletException if the HTTP request cannot be handled or if either parameter is not an instance of its
     * respective {@link HttpServletRequest} or {@link HttpServletResponse} counterparts.
     * 
     * @see jakarta.servlet.Servlet#service
     */
    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
        HttpServletRequest request;
        HttpServletResponse response;

        if (!(req instanceof HttpServletRequest && res instanceof HttpServletResponse)) {
            throw new ServletException("non-HTTP request or response");
        }

        request = (HttpServletRequest) req;
        response = (HttpServletResponse) res;

        service(request, response);
    }
}
```

继续阅读。
```java
  protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                if (ifModifiedSince < lastModified) {
                    // If the servlet mod time is later, call doGet()
                    // Round down to the nearest second for a proper compare
                    // A ifModifiedSince of -1 will always be less
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }

        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);

        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);

        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);

        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);

        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req, resp);

        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req, resp);

        } else {
            //
            // Note that this means NO servlet supports whatever
            // method was requested, anywhere on this server.
            //

            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```

这里按Http Method分别请求对应的方法，我们选择doPost()方法进入FrameworkServlet的doPost()方法。
```java
	/**
	 * Delegate POST requests to {@link #processRequest}.
	 * @see #doService
	 */
	@Override
	protected final void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}
```

这里是统一的处理HttpServletRequest的方法:

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;
            //获得之前请求的LocaleContext
		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContext localeContext = buildLocaleContext(request);
            //获得之前请求的参数
		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
            //初始化ContextHolders
		initContextHolders(request, localeContext, requestAttributes);

		try {
            //处理请求的方法
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new ServletException("Request processing failed: " + ex, ex);
		}

		finally {
            //重置ContextHolders
			resetContextHolders(request, previousLocaleContext, previousAttributes);
			if (requestAttributes != null) {
				requestAttributes.requestCompleted();
			}
            //纪录日志和发布事件
			logResult(request, response, failureCause, asyncManager);
			publishRequestHandledEvent(request, response, startTime, failureCause);
		}
	}

```

继续深入，来到DispatcherServlet的doService方法。

```java
@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
                 //将WebApplicationContext、localeResolver、themeResolver等资源的引用存入reques的参数内
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		RequestPath previousRequestPath = null;
		if (this.parseRequestPath) {
			previousRequestPath = (RequestPath) request.getAttribute(ServletRequestPathUtils.PATH_ATTRIBUTE);
			ServletRequestPathUtils.parseAndCache(request);
		}

		try {
                    //真正的处理请求方法
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
			if (this.parseRequestPath) {
				ServletRequestPathUtils.setParsedRequestPath(previousRequestPath, request);
			}
		}
	}
```
通过层层转发，我们最终找到了真正的处理请求方法doDispatch()。
## 2.doDispatch()方法

这里就是DispatcherServlet处理Http请求的核心方法了。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
                             //请求HandlerMapping获得HandlerExecutionChain(handler+interceptorList)
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
                            //获得handler的适配器HandlerAdapter
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = HttpMethod.GET.matches(method);
				if (isGet || HttpMethod.HEAD.matches(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
                            //拦截器前置处理
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
                            //调用handler(controller)的目标方法
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
                            //拦截器后置处理
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new ServletException("Handler dispatch failed: " + err, err);
			}
                            //处理handler(controller)的目标方法的返回结果
                            //1.异常处理processHandlerException
                            //2.视图解析resolveViewName
                            //3.视图渲染view.render()
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
                         //拦截器最终处理
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new ServletException("Handler processing failed: " + err, err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

可以看到整个流程解读还是很简单，没有很高的复杂度，也没有非常多的分支。<br>
下面简单阅读一下这些主要的处理方法。

TODO~
### 2.1 getHandler()方法

### 2.2 applyPreHandle()方法

### 2.3 HandlerAdapter.handle()方法

### 2.4 applyPostHandle()方法

### 2.5 processDispatchResult()方法

### 2.6 resolveViewName()方法

### 2.7 View.render()方法

### 2.8 triggerAfterCompletion()方法


# 三、拦截器

## 1.拦截器的简单测试

## 2.过滤器、拦截器、自定义AOP切面三者的执行顺序


# 四、@ResponseBody的处理流程
此节在这里，这里会不会太长了，单独拆分为一篇文章？

# 五、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)<br>