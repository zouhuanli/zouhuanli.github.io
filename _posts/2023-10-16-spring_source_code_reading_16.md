---
layout: post
title: "Spring源码阅读十六:@RequestBody和@ResponseBody的处理流程"
date: 2023-10-16
tags: [ spring ]
comments: true
author: zouhuanli
---

本文是Spring源码阅读计划的第十六篇文章，本文着重介绍@RequestBody和@ResponseBody的处理流程。<br>
上一篇文章已经介绍DispatcherServlet的请求处理流程流程，@ResponseBody主要用于不返回View直接写回数据。在前后端分离的架构，<br>
后端不负责视图和视图渲染这些了，只提供数据交互的API，这也是我单独开一篇文讲解这两个注解的原因，笔者本想上一篇文章一起探讨这个话题的。
本文源码地址为:[https://github.com/zouhuanli/SpringMvcDemo.git](https://github.com/zouhuanli/SpringMvcDemo.git).<br>

# 一、@RequestBody注解处理流程

先看下http请求的格式。

```text
POST ---方法 http://localhost:8080/springmvcdemo_war/user/hello ---服务地址  HTTP/1.1 ---协议版本号
content-type: application/json  ---报文头
charset: utf-8
Content-Length: 51
Connection: Keep-Alive
User-Agent: Apache-HttpClient/4.5.14 (Java/17.0.8.1)
Cookie: JSESSIONID=C9E9B60D886134D3DA1698D8B4667C66
Accept-Encoding: br,deflate,gzip,x-gzip
 ---回车
{---报文体
  "name": "李四",
  "age": 19,
  "id": "1111"
}
```

RequestBody注解是将报文体绑定为方法的参数对象。例如：

```java
 /**
     * 返回Json数据示例
     *
     * @param requestBody
     * @param request
     * @param response
     * @return
     */
    @RequestMapping(value = "/hello", method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseBody
    public ResponseEntity<String> hello(@RequestBody String requestBody, HttpServletRequest request, HttpServletResponse response) {
        LOGGER.info("receive requestBody:{} ", requestBody);
        String responseJson = "{\"userName\":\"张三\",\"age\":18}";
        return new ResponseEntity<>(responseJson, HttpStatus.OK);
    }
```
看下源码的注释解释。
```text
 * Annotation indicating a method parameter should be bound to the body of the web request.
 * The body of the request is passed through an {@link HttpMessageConverter} to resolve the
 * method argument depending on the content type of the request. Optionally, automatic
 * validation can be applied by annotating the argument with {@code @Valid}.
```

将报文体绑定为方法的参数对象，使用HttpMessageConverter将请求体转换为方法参数，使用@Valid注解进行校验请求。<br>
我们接着上文从“mv = ha.handle(processedRequest, response, mappedHandler.getHandler())”进入InvocableHandlerMethod#invokeForRequest方法开始阅读。

```java
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        //获得请求参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
        //do开头的，真正的调用目标方法
		return doInvoke(args);
	}
```

## 1.getMethodArgumentValues方法

我们进入getMethodArgumentValues方法。

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
                        //原始的请求参数列表，本例中的就是UserController#hello方法的(@RequestBody String requestBody, HttpServletRequest request, HttpServletResponse response)三个参数
		MethodParameter[] parameters = getMethodParameters();

		Object[] args = new Object[parameters.length];
                        //遍历处理方法的每个参数，我们这里的index=0的参数有RequestBody注解。
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			
		}
		return args;
	}
```

进入HandlerMethodArgumentResolverComposite#supportsParameter方法。

```java
                    //查找匹配的参数解析器
@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return getArgumentResolver(parameter) != null;
	}
```

继续深入。


```java
                     //从参数解析器中获取匹配的解析器
@Nullable
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
				if (resolver.supportsParameter(parameter)) {
					result = resolver;
                    //存入缓存集合
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```

解析器列表如图。

![argumentResolvers](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-16-spring_source_code_reading_16/argumentResolvers.png)

我们这里自己的应用没有注册方法参数解析器，那么是框架自己注册和初始化的。
我们跟踪引用，发现在RequestMappingHandlerAdapter#afterPropertiesSet方法中有注册参数解析器。

```java
@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();
		initMessageConverters();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
```
```java
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
		if (KotlinDetector.isKotlinPresent()) {
			resolvers.add(new ContinuationHandlerMethodArgumentResolver());
		}

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new PrincipalMethodArgumentResolver());
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}

```
结果，去掉两个if条件，正好27个解析器,type也对的上的。<br>
这里的处理就是很简单，遍历27个注册的参数解析器，返回第一个匹配的，也就是supportsParameter方法返回true的。<br>
最后来到本例工程匹配的RequestResponseBodyMethodProcessor#supportsParameter方法，这里就简单判断了一下是否包含RequestBody注解。
```java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}
```
最终( String requestBody)参数的解析器是RequestResponseBodyMethodProcessor。

解析器匹配完成了，接下来进入resolveArgument方法。

### 1.1 HandlerMethodArgumentResolver#resolveArgument方法

```java
@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		parameter = parameter.nestedIfOptional();
        //读写Http请求，转换为对应对象
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
		String name = Conventions.getVariableNameForParameter(parameter);

		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
			if (arg != null) {
				validateIfApplicable(binder, parameter);
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
				}
			}
			if (mavContainer != null) {
				mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
			}
		}
                        //适配
		return adaptArgumentIfNecessary(arg, parameter);
	}
```

继续进入readWithMessageConverters方法。

```java
@Override
	protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter,
			Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
		Assert.state(servletRequest != null, "No HttpServletRequest");
		ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);

		Object arg = readWithMessageConverters(inputMessage, parameter, paramType);
		if (arg == null && checkRequired(parameter)) {
			throw new HttpMessageNotReadableException("Required request body is missing: " +
					parameter.getExecutable().toGenericString(), inputMessage);
		}
		return arg;
	}
```

继续深入。
```java
@Nullable
	@SuppressWarnings({ "unchecked", "rawtypes" })
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		Class<?> contextClass = parameter.getContainingClass();
		Class<T> targetClass = (targetType instanceof Class clazz ? clazz : null);
		if (targetClass == null) {
			ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
			targetClass = (Class<T>) resolvableType.resolve();
		}

		MediaType contentType;
		boolean noContentType = false;
		try {
                      //获得contentType
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(
					ex.getMessage(), getSupportedMediaTypes(targetClass != null ? targetClass : Object.class));
		}
		if (contentType == null) {
			noContentType = true;
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}

		HttpMethod httpMethod = (inputMessage instanceof HttpRequest httpRequest ? httpRequest.getMethod() : null);
		Object body = NO_VALUE;

		EmptyBodyCheckingHttpInputMessage message = null;
		try {
			message = new EmptyBodyCheckingHttpInputMessage(inputMessage);

			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ghmc ? ghmc : null);
				if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					if (message.hasBody()) {
                        //切面通知的前置处理
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
                        //真正的读取方法
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
                        //切面通知的后置处理
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					else {
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					break;
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("I/O error while reading input message", ex, inputMessage);
		}
		finally {
			if (message != null && message.hasBody()) {
				closeStreamIfNecessary(message.getBody());
			}
		}

		if (body == NO_VALUE) {
			if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
					(noContentType && !message.hasBody())) {
				return null;
			}
			throw new HttpMediaTypeNotSupportedException(contentType,
					getSupportedMediaTypes(targetClass != null ? targetClass : Object.class), httpMethod);
		}

		MediaType selectedContentType = contentType;
		Object theBody = body;
		LogFormatUtils.traceDebug(logger, traceOn -> {
			String formatted = LogFormatUtils.formatValue(theBody, !traceOn);
			return "Read \"" + selectedContentType + "\" to [" + formatted + "]";
		});

		return body;
	}
```
我们看下Http请求消息转换器的实例。

![messageConverters](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-16-spring_source_code_reading_16/messageConverters.png)

这里得到的args[0]就是一个json字符串。

![args](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-16-spring_source_code_reading_16/args.png)

我们再修改一下UserController#方法，修改为如下：

```java
@RequestMapping(value = "/hello", method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_VALUE)
@ResponseBody
public ResponseEntity<User> hello(@RequestBody User requestBody, HttpServletRequest request, HttpServletResponse response) {
        LOGGER.info("receive requestBody:{} ", requestBody);
        String responseJson = "{\"userName\":\"张三\",\"age\":18}";
        return new ResponseEntity<>(requestBody, HttpStatus.OK);
        }
```

再发送POST请求。

```text
POST  http://localhost:8080/springmvcdemo_war/user/hello  HTTP/1.1
content-type: application/json
charset: utf-8

{
  "uname": "李四",
  "age": 19,
  "uid": "1111"
}
```

再进入RequestResponseBodyMethodProcessor#readWithMessageConverters方法。这里的消息转换器是“MappingJackson2HttpMessageConverter”，而不是StringHttpMessageConverter。
我们继续进入来到read方法。


### 1.2 HttpMessageConverter#read方法

进入read方法：
```java
@Override
	public final T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {

		return readInternal(clazz, inputMessage);
	}
```

继续进入AbstractJackson2HttpMessageConverter这个json消息转换器。其read方法如下：

```java
	@Override
public Object read(Type type, @Nullable Class<?> contextClass, HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException {

        JavaType javaType = getJavaType(type, contextClass);
        return readJavaType(javaType, inputMessage);
        }
```

这里做了反序列化，将Json字符串转换为User对象了。

```java
private Object readJavaType(JavaType javaType, HttpInputMessage inputMessage) throws IOException {
		MediaType contentType = inputMessage.getHeaders().getContentType();
		Charset charset = getCharset(contentType);

		ObjectMapper objectMapper = selectObjectMapper(javaType.getRawClass(), contentType);
		Assert.state(objectMapper != null, () -> "No ObjectMapper for " + javaType);

		boolean isUnicode = ENCODINGS.containsKey(charset.name()) ||
				"UTF-16".equals(charset.name()) ||
				"UTF-32".equals(charset.name());
		try {
			InputStream inputStream = StreamUtils.nonClosing(inputMessage.getBody());
			if (inputMessage instanceof MappingJacksonInputMessage mappingJacksonInputMessage) {
				Class<?> deserializationView = mappingJacksonInputMessage.getDeserializationView();
				if (deserializationView != null) {
					ObjectReader objectReader = objectMapper.readerWithView(deserializationView).forType(javaType);
					objectReader = customizeReader(objectReader, javaType);
					if (isUnicode) {
						return objectReader.readValue(inputStream);
					}
					else {
						Reader reader = new InputStreamReader(inputStream, charset);
                        //反序列化为User对象
						return objectReader.readValue(reader);
					}
				}
			}

			ObjectReader objectReader = objectMapper.reader().forType(javaType);
			objectReader = customizeReader(objectReader, javaType);
			if (isUnicode) {
				return objectReader.readValue(inputStream);
			}
			else {
				Reader reader = new InputStreamReader(inputStream, charset);
				return objectReader.readValue(reader);
			}
		}
		catch (InvalidDefinitionException ex) {
			throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
		}
		catch (JsonProcessingException ex) {
			throw new HttpMessageNotReadableException("JSON parse error: " + ex.getOriginalMessage(), ex, inputMessage);
		}
	}

```

最终的body(User对象)就是这样了。

```java
body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
```
![body](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-16-spring_source_code_reading_16/body.png)


## 2.doInvoke方法
我们继续阅读doInvoke方法。
```java
	@Nullable
	protected Object doInvoke(Object... args) throws Exception{
        Method method=getBridgedMethod();
        try{
        if(KotlinDetector.isSuspendingFunction(method)){
        return invokeSuspendingFunction(method,getBean(),args);
        }
        return method.invoke(getBean(),args);
        }
        }
```
本例工程中我们得到一个方法签名是“public org.springframework.http.ResponseEntity com.homura.controller.UserController<br>.hello(java.lang.String,jakarta.servlet.http.HttpServletRequest,jakarta.servlet.http.HttpServletResponse)”
的桥连方法。getBean则是""class com.homura.controller.UserController$$SpringCGLIB$$1"这样的原始对象是UserController的代理对象。<br>
然后通过反射调用invoke调用controller的处理方法，得到处理结果。



到这，Http请求的参数转换就解读完成了。

# 二、@ResponseBody注解处理流程

我们先从ServletInvocableHandlerMethod#invokeAndHandle方法读起。

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
            //返回值类型处理器
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
```
## 1.HandlerMethodReturnValueHandler#handleReturnValue方法

returnValueHandlers返回值处理器的注册也是在之前的afterPropertiesSet方法。<br>
先匹配到对应的returnValueHandler,本例工程中是HttpEntityMethodProcessor或RequestResponseBodyMethodProcessor：

```java
@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}
```
我们继续跟踪执行代码，来到HttpEntityMethodProcessor#handleReturnValue方法。

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		mavContainer.setRequestHandled(true);
		if (returnValue == null) {
			return;
		}

		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		HttpEntity<?> httpEntity;
		if (returnValue instanceof ErrorResponse response) {
			httpEntity = new ResponseEntity<>(response.getBody(), response.getHeaders(), response.getStatusCode());
		}
		else if (returnValue instanceof ProblemDetail detail) {
			httpEntity = ResponseEntity.of(detail).build();
		}
		else {
			Assert.isInstanceOf(HttpEntity.class, returnValue);
			httpEntity = (HttpEntity<?>) returnValue;
		}

		if (httpEntity.getBody() instanceof ProblemDetail detail) {
			if (detail.getInstance() == null) {
				URI path = URI.create(inputMessage.getServletRequest().getRequestURI());
				detail.setInstance(path);
			}
			if (logger.isWarnEnabled() && httpEntity instanceof ResponseEntity<?> responseEntity) {
				if (responseEntity.getStatusCode().value() != detail.getStatus()) {
					logger.warn(returnType.getExecutable().toGenericString() +
							" returned ResponseEntity: " + responseEntity + ", but its status" +
							" doesn't match the ProblemDetail status: " + detail.getStatus());
				}
			}
		}

		HttpHeaders outputHeaders = outputMessage.getHeaders();
		HttpHeaders entityHeaders = httpEntity.getHeaders();
		if (!entityHeaders.isEmpty()) {
			entityHeaders.forEach((key, value) -> {
				if (HttpHeaders.VARY.equals(key) && outputHeaders.containsKey(HttpHeaders.VARY)) {
					List<String> values = getVaryRequestHeadersToAdd(outputHeaders, entityHeaders);
					if (!values.isEmpty()) {
						outputHeaders.setVary(values);
					}
				}
				else {
					outputHeaders.put(key, value);
				}
			});
		}

		if (httpEntity instanceof ResponseEntity<?> responseEntity) {
			int returnStatus = responseEntity.getStatusCode().value();
			outputMessage.getServletResponse().setStatus(returnStatus);
			if (returnStatus == 200) {
				HttpMethod method = inputMessage.getMethod();
				if ((HttpMethod.GET.equals(method) || HttpMethod.HEAD.equals(method))
						&& isResourceNotModified(inputMessage, outputMessage)) {
					outputMessage.flush();
					return;
				}
			}
			else if (returnStatus / 100 == 3) {
				String location = outputHeaders.getFirst("location");
				if (location != null) {
					saveFlashAttributes(mavContainer, webRequest, location);
				}
			}
		}

		// Try even with null body. ResponseBodyAdvice could get involved.
        //注意看这行，写入http响应报文outputMessage
        writeWithMessageConverters(httpEntity.getBody(), returnType, inputMessage, outputMessage);

		// Ensure headers are flushed even if no body was written.
		outputMessage.flush();
	}
```
返回的httpEntity如下：

![httpEntity](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2023-10-16-spring_source_code_reading_16/httpEntity.png)

我们进入writeWithMessageConverters方法：

```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		Object body;
		Class<?> valueType;
		Type targetType;

		if (value instanceof CharSequence) {
			body = value.toString();
			valueType = String.class;
			targetType = String.class;
		}
		else {
			body = value;
                     //responseBody数据的类型
			valueType = getReturnValueType(body, returnType);
			targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
		}

		if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}

		MediaType selectedMediaType = null;
		MediaType contentType = outputMessage.getHeaders().getContentType();
		boolean isContentTypePreset = contentType != null && contentType.isConcrete();
		if (isContentTypePreset) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found 'Content-Type:" + contentType + "' in response");
			}
			selectedMediaType = contentType;
		}
		else {
			HttpServletRequest request = inputMessage.getServletRequest();
			List<MediaType> acceptableTypes;
			try {
				acceptableTypes = getAcceptableMediaTypes(request);
			}
			catch (HttpMediaTypeNotAcceptableException ex) {
				int series = outputMessage.getServletResponse().getStatus() / 100;
				if (body == null || series == 4 || series == 5) {
					if (logger.isDebugEnabled()) {
						logger.debug("Ignoring error response content (if any). " + ex);
					}
					return;
				}
				throw ex;
			}
                        //通过messageConverters匹配返回的media-type，这里是"application/json"
			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
			if (body != null && producibleTypes.isEmpty()) {
				throw new HttpMessageNotWritableException(
						"No converter found for return value of type: " + valueType);
			}

			List<MediaType> compatibleMediaTypes = new ArrayList<>();
			determineCompatibleMediaTypes(acceptableTypes, producibleTypes, compatibleMediaTypes);

			// For ProblemDetail, fall back on RFC 7807 format
			if (compatibleMediaTypes.isEmpty() && ProblemDetail.class.isAssignableFrom(valueType)) {
				determineCompatibleMediaTypes(this.problemMediaTypes, producibleTypes, compatibleMediaTypes);
			}

			if (compatibleMediaTypes.isEmpty()) {
				if (logger.isDebugEnabled()) {
					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
				}
				if (body != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
				}
				return;
			}

			MimeTypeUtils.sortBySpecificity(compatibleMediaTypes);

			for (MediaType mediaType : compatibleMediaTypes) {
				if (mediaType.isConcrete()) {
					selectedMediaType = mediaType;
					break;
				}
				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
					break;
				}
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Using '" + selectedMediaType + "', given " +
						acceptableTypes + " and supported " + producibleTypes);
			}
		}

		if (selectedMediaType != null) {
                         //消息转换器的列表匹配合适的转换器
			selectedMediaType = selectedMediaType.removeQualityValue();
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				GenericHttpMessageConverter genericConverter =
						(converter instanceof GenericHttpMessageConverter ghmc ? ghmc : null);
                            //注意看canWrite方法和write方法。这里匹配的是MappingJackson2HttpMessageConverter
				if (genericConverter != null ?
						((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {
					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					if (body != null) {
						Object theBody = body;
						LogFormatUtils.traceDebug(logger, traceOn ->
								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
						addContentDispositionHeader(inputMessage, outputMessage);
						if (genericConverter != null) {
                            //序列化User为Json字符串的关键代码
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
						}
					}
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Nothing to write: null body");
						}
					}
					return;
				}
			}
		}

		if (body != null) {
			Set<MediaType> producibleMediaTypes =
					(Set<MediaType>) inputMessage.getServletRequest()
							.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

			if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
				throw new HttpMessageNotWritableException(
						"No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
			}
			throw new HttpMediaTypeNotAcceptableException(getSupportedMediaTypes(body.getClass()));
		}
	}
```

## 2.HttpMessageConverter#write方法

我们继续进入write方法。

```java
@Override
	public final void write(final T t, @Nullable final Type type, @Nullable MediaType contentType,
			HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {

		final HttpHeaders headers = outputMessage.getHeaders();
		addDefaultHeaders(headers, t, contentType);

		if (outputMessage instanceof StreamingHttpOutputMessage streamingOutputMessage) {
			streamingOutputMessage.setBody(outputStream -> writeInternal(t, type, new HttpOutputMessage() {
				@Override
				public OutputStream getBody() {
					return outputStream;
				}
				@Override
				public HttpHeaders getHeaders() {
					return headers;
				}
			}));
		}
		else {
			writeInternal(t, type, outputMessage);
			outputMessage.getBody().flush();
		}
	}
```

再来到AbstractJackson2HttpMessageConverter#writeInternal方法,这里就是序列化User对象为Json并写入HttpOutputMessage中。

```java

	@Override
	protected void writeInternal(Object object, @Nullable Type type, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		MediaType contentType = outputMessage.getHeaders().getContentType();
		JsonEncoding encoding = getJsonEncoding(contentType);

		Class<?> clazz = (object instanceof MappingJacksonValue mappingJacksonValue ?
				mappingJacksonValue.getValue().getClass() : object.getClass());
		ObjectMapper objectMapper = selectObjectMapper(clazz, contentType);
		Assert.state(objectMapper != null, () -> "No ObjectMapper for " + clazz.getName());

		OutputStream outputStream = StreamUtils.nonClosing(outputMessage.getBody());
		try (JsonGenerator generator = objectMapper.getFactory().createGenerator(outputStream, encoding)) {
			writePrefix(generator, object);

			Object value = object;
			Class<?> serializationView = null;
			FilterProvider filters = null;
			JavaType javaType = null;

			if (object instanceof MappingJacksonValue mappingJacksonValue) {
				value = mappingJacksonValue.getValue();
				serializationView = mappingJacksonValue.getSerializationView();
				filters = mappingJacksonValue.getFilters();
			}
			if (type != null && TypeUtils.isAssignable(type, value.getClass())) {
				javaType = getJavaType(type, null);
			}

			ObjectWriter objectWriter = (serializationView != null ?
					objectMapper.writerWithView(serializationView) : objectMapper.writer());
			if (filters != null) {
				objectWriter = objectWriter.with(filters);
			}
			if (javaType != null && (javaType.isContainerType() || javaType.isTypeOrSubTypeOf(Optional.class))) {
				objectWriter = objectWriter.forType(javaType);
			}
			SerializationConfig config = objectWriter.getConfig();
			if (contentType != null && contentType.isCompatibleWith(MediaType.TEXT_EVENT_STREAM) &&
					config.isEnabled(SerializationFeature.INDENT_OUTPUT)) {
				objectWriter = objectWriter.with(this.ssePrettyPrinter);
			}
			objectWriter = customizeWriter(objectWriter, javaType, contentType);
			objectWriter.writeValue(generator, value);

			writeSuffix(generator, object);
			generator.flush();
		}
		catch (InvalidDefinitionException ex) {
			throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
		}
		catch (JsonProcessingException ex) {
			throw new HttpMessageNotWritableException("Could not write JSON: " + ex.getOriginalMessage(), ex);
		}
	}
```

注意看HttpEntityMethodProcessor#handleReturnValue方法内的：
```java

		mavContainer.setRequestHandled(true);
		if (returnValue == null) {
			return;
		}
```

和RequestMappingHandlerAdapter#getModelAndView方法内的：
```java

		modelFactory.updateModel(webRequest, mavContainer);
		if (mavContainer.isRequestHandled()) {
			return null;
		}
```
这里设置了写入响应对象之后，没有返回View对象给HandlerAdapter的handle方法，直接返回了null，ModeAndView对象为null,后面的视图解析和视图渲染都不做了。


```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

我们修改hello方法的返回参数类型是User：
```java
 @RequestMapping(value = "/hello", method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseBody
    public User hello(@RequestBody User requestBody, HttpServletRequest request, HttpServletResponse response) {
        LOGGER.info("receive requestBody:{} ", requestBody);
        String responseJson = "{\"userName\":\"张三\",\"age\":18}";
        return requestBody;
    }
```
跟踪执行流程发现returnValueHandlers匹配的返回值处理器是RequestResponseBodyMethodProcessor。<br>
依旧是AbstractMessageConverterMethodProcessor#writeWithMessageConverters选择对应的消息转换器，这里依旧是MappingJackson2HttpMessageConverter,
在”genericConverter.write(body, targetType, selectedMediaType, outputMessage);“序列化User对象为Json，并写入到Response对象中。<br>

可以看到HandlerMethodReturnValueHandler#handleReturnValue方法主要是设置是否需要View，和将Controller/handler的方法的执行结果设置为ModelAndView或者直接转换为Http请求结果写入到Response响应中。


# 四、参考材料

1.Spring源码(版本6.0.11)<br>
2.《spring源码深度解析》(郝佳)<br>
3.《Spring Framework Documentation》(Version 6.0.8)<br>