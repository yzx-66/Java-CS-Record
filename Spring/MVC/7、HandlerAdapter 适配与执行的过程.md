# 前言
## 三个核心接口
HttpMessageConverter

* SpringMVC处理请求和响应时，支持多种类型的请求参数和返回类型，而此种功能的实现就需要对HTTP消息体和参数及返回值进行转换，为此SpringMVC提供了大量的转换类，**所有转换类都实现了HttpMessageConverter接口**。
* HttpMessageConverter接口定义了5个方法，用于将HTTP请求报文转换为java对象，以及将java对象转换为HTTP响应报文。对应到SpringMVC的Controller方法，read方法即是读取HTTP请求转换为参数对象，write方法即是将返回值对象转换为HTTP响应报文。

```javascript
public interface HttpMessageConverter<T> {

    // 当前转换器是否能将HTTP报文转换为对象类型
    boolean canRead(Class<?> clazz, MediaType mediaType);

    // 当前转换器是否能将对象类型转换为HTTP报文
    boolean canWrite(Class<?> clazz, MediaType mediaType);

    // 转换器能支持的HTTP媒体类型
    List<MediaType> getSupportedMediaTypes();

    // 转换HTTP报文为特定类型
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException;

    // 将特定类型对象转换为HTTP报文
    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException;

}
```



HandlerMethodArgumentResolver 与 HandlerMethodReturnValueHandler
* 参数解析器HandlerMethodArgumentResolver
* 返回值处理器HandlerMethodReturnValueHandler。
* 参数解析器和返回值处理器在底层处理时，**内部都是通过HttpMessageConverter进行转换**。

```javascript
// 参数解析器接口
public interface HandlerMethodArgumentResolver {

    // 解析器是否支持方法参数
    boolean supportsParameter(MethodParameter parameter);

    // 解析HTTP报文中对应的方法参数
    Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;

}

// 返回值处理器接口
public interface HandlerMethodReturnValueHandler {

    // 处理器是否支持返回值类型
    boolean supportsReturnType(MethodParameter returnType);

    // 将返回值解析为HTTP响应报文
    void handleReturnValue(Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}
```

## RequestMappingHandlerAdapter 对上面三个核心对象的初始化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222202219724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)
初始化默认 HttpMessageConverters
```java
public RequestMappingHandlerAdapter() {
        StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
        stringHttpMessageConverter.setWriteAcceptCharset(false);
        this.messageConverters = new ArrayList(4);
        this.messageConverters.add(new ByteArrayHttpMessageConverter());
        this.messageConverters.add(stringHttpMessageConverter);
        this.messageConverters.add(new SourceHttpMessageConverter());
        this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
    }
```

初始化默认argumentResolvers 与 returnValueHandlers 
```java
public void afterPropertiesSet() {
        this.initControllerAdviceCache();
        List handlers;
        if (this.argumentResolvers == null) {
            handlers = this.getDefaultArgumentResolvers();
            this.argumentResolvers = (new HandlerMethodArgumentResolverComposite()).addResolvers(handlers);
        }

        if (this.initBinderArgumentResolvers == null) {
            handlers = this.getDefaultInitBinderArgumentResolvers();
            this.initBinderArgumentResolvers = (new HandlerMethodArgumentResolverComposite()).addResolvers(handlers);
        }

        if (this.returnValueHandlers == null) {
            handlers = this.getDefaultReturnValueHandlers();
            this.returnValueHandlers = (new HandlerMethodReturnValueHandlerComposite()).addHandlers(handlers);
        }

    }

```

圈起来的 RequestResponBodyMethodProcessor 就是处理 @RequestBody 与 @ResponBody 的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222202454125.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222202537604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

# 二、@RequestBody解析过程
## ServletInvocableHandlerMethod
所有的http请求都会进入ServletInvocableHandlerMethod类（继承InvocableHandlerMethod，所有的参数解析器都会在在这里面进行初始化）的invokeAndHandle方法中，我们来具体看看invokeAndHandle方法是干什么的。
### invokeAndHandle

```javascript
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
    // 执行http请求
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
	// 返回值处理
	try {
		this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}catch (Exception ex) {
		if (logger.isTraceEnabled()){
		logger.trace(formatErrorForReturnValue(returnValue), ex);
		}
		throw ex;
	}
}
```

我们可以看到invokeAndHandle方法都会进入invokeForRequest方法中，invokeForRequest方法就是实现@RequestBody注解的功能，将http请求报文解析为我们设置的对象。我们进入该方法看看，里面具体做了哪些事情。

### invokeForRequest

```javascript
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
    // http报文解析为对象数组
	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		logger.trace("Arguments: " + Arrays.toString(args));
	}
	//执行@PostMapping、@GetMapping等接口
	return doInvoke(args);
}
```

我们可以看到invokeForRequest中主要做了两件事情，一个是通过getMethodArgumentValues方法返回http解析后的对象数组，然后通过doInvoke方法执行接口的具体业务逻辑代码。

我们接着进入getMethodArgumentValues方法，细看一下@RequestBody的具体解析过程。

### getMethodArgumentValues
```javascript
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
    
    // 获取http请求参数
	MethodParameter[] parameters = getMethodParameters();
	if (ObjectUtils.isEmpty(parameters)) {
		return EMPTY_ARGS;
	}
	Object[] args = new Object[parameters.length];
	// 遍历所有参数，挨个解析
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		args[i] = findProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (!this.resolvers.supportsParameter(parameter)) {
			throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
		}
		try {
		    // 参数解析器解析HTTP报文
			args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
		}catch (Exception ex) {
			// Leave stack trace for later, exception may actually be resolved and handled...
			if (logger.isDebugEnabled()) {
				String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
					logger.debug(formatArgumentError(parameter, exMsg));
				}
			}
			throw ex;
		}
	}
	return args;
}
```


其中 `this.resolvers.supportsParameter(parameter)` 用来判断请求参数是否合法，`this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory)` 方法最终实现@RequestBody解析操作。
我们来看看 `this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory)`中做了什么。

## RequestResponBodyMethodProcessor
### resolveArgument
```javascript
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    // 获取对应的解析器
	HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
	if (resolver == null) {
		throw new IllegalArgumentException(
					"Unsupported parameter type [" + parameter.getParameterType().getName() + "]." +
							" supportsParameter should be called first.");
	}
	// 通过HandlerMethodArgumentResolver 解析器解析http报文
	return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

getArgumentResolver 方法来获取对应的 HandlerMethodArgumentResolver 参数解析器，参数解析器最终通过RequestResponseBodyMethodProcessor 类来具体执行解析过程
我们接着来看看RequestResponseBodyMethodProcessor中resolveArgument方法又是怎样的一个处理过程。

>  不同的resolvers（HandlerMethodArgumentResolver接口）会对应不同的参数解析器
>  例如public String testDemo(String name)，解析器就会变成ServletRequestMethodArgumentResolver，如果是@RequestBody，参数解析器就是RequestResponseBodyMethodProcessor 

### resolveArgument

```javascript
@Override
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

	parameter = parameter.nestedIfOptional();
	// 通过HttpMessageConverter来解析http报文为Object对象
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
	return adaptArgumentIfNecessary(arg, parameter);
}
```

readWithMessageConverters方法中，HttpMessageConverter（接口对应实现类）的read方法实现了http报文解析，我们来看看最终http参数解析部分的代码。

## readWithMessageConverters
```javascript
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

	....
		
	try {
		message = new EmptyBodyCheckingHttpInputMessage(inputMessage);

		for (HttpMessageConverter<?> converter : this.messageConverters) {
			Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
			GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
			// 判断转换器是否支持参数类型
			if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
				if (message.hasBody()) {
					HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
					// read方法执行HTTP报文到参数的转换
					body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
					body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
				}else {
					body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
				}
				break;
			}
		}
	}catch (IOException ex) {
		throw new HttpMessageNotReadableException("I/O error while reading input message", ex, inputMessage);
	}

	....
}
```

代码部分省略了，关键部分即是遍历所有的HttpMessageConverter，然后通过canRead方法判断解析器是否支持，最后执行AbstractJackson2HttpMessageConverter对象（HttpMessageConverter实现类）的read方法完成最后的参数解析。

>  AbstractJackson2HttpMessageConverter对象的read方法，核心是利用了jackson工具，将http报文的json字符串转换为object对象并返回。 



# 三、@ResponseBody返回值序列化过程
## ServletInvocableHandlerMethod
执行完doInvoke逻辑代码之后，通过ServletInvocableHandlerMethod对象的invokeAndHandle方法，利用返回值处理器对返回值进行序列化输出。

```java
this.returnValueHandlers.handleReturnValue(returnValue, 
			getReturnValueType(returnValue), mavContainer, webRequest);
```

returnValueHandlers为HandlerMethodReturnValueHandlerComposite对象，该对象实现了HandlerMethodReturnValueHandler接口，我们接着来看看handleReturnValue方法的具体实现。


### handleReturnValue

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        // 选择合适的HandlerMethodReturnValueHandler返回值处理器
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		// 执行返回值处理
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}
```

selectHandler方法会提供合适的HandlerMethodReturnValueHandler，用来处理返回值。


我们看到的HandlerMethodReturnValueHandler处理器最终也是由RequestResponseBodyMethodProcessor实现的，我们具体来看看handleReturnValue方法。

>  handler（HandlerMethodReturnValueHandler）接口会根据不同类型选择不同的返回值处理器
>  例如页面跳转类型的处理器就是ViewNameMethodReturnValueHandler。 

## RequestResponBodyMethodProcessor
### handleReturnValue
```javascript
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		mavContainer.setRequestHandled(true);
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// 调用HttpMessageConverter执行
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
	
```
### createOutputMessage
大家都知道@ResponseBody需要通过io流来读取，也就HttpMessageConverter最终的write会写入到io输出流中，上面的createOutputMessage(webRequest)方法就是创建一个输出流，我们来具体看看它的实现。

```javascript
protected ServletServerHttpResponse createOutputMessage(NativeWebRequest webRequest) {
	HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
		Assert.state(response != null, "No HttpServletResponse");
		return new ServletServerHttpResponse(response);
}
	

public class ServletServerHttpResponse implements ServerHttpResponse {

	private final HttpServletResponse servletResponse;

	private final HttpHeaders headers;

	private boolean headersWritten = false;

	private boolean bodyUsed = false;


	/**
	 * Construct a new instance of the ServletServerHttpResponse based on the given {@link HttpServletResponse}.
	 * @param servletResponse the servlet response
	 */
	public ServletServerHttpResponse(HttpServletResponse servletResponse) {
		Assert.notNull(servletResponse, "HttpServletResponse must not be null");
		this.servletResponse = servletResponse;
		this.headers = new ServletResponseHttpHeaders();
	}
}

public interface ServletResponse {
    String getCharacterEncoding();

    String getContentType();

    ServletOutputStream getOutputStream() throws IOException;

    PrintWriter getWriter() throws IOException;

    void setCharacterEncoding(String var1);

    void setContentLength(int var1);

    void setContentLengthLong(long var1);

    void setContentType(String var1);

    void setBufferSize(int var1);

    int getBufferSize();

    void flushBuffer() throws IOException;

    void resetBuffer();

    boolean isCommitted();

    void reset();

    void setLocale(Locale var1);

    Locale getLocale();
}	
```

createOutputMessage方法中创建了ServletServerHttpResponse ，然后通过 ((HttpMessageConverter) messageConverter).write(outputValue, selectedMediaType, outputMessage)方法写入到输出流中。write方法的核心也是通过Jackson工具将对象解析为json字符串。
### writeWithMessageConverters
```	java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
	....
	    for (HttpMessageConverter<?> converter : this.messageConverters) {
				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
						(GenericHttpMessageConverter<?>) converter : null);
				// 判断是否支持返回值类型，返回值类型很有可能不同，如String，Map，List等
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
						    // 执行返回值转换
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
						    // 执行返回值转换
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
	
	....
}
```

我们看到最终还是由HttpMessageConverter（AbstractGenericHttpMessageConverter实现类）的write方法来进行对象的序列化输出。

## AbstractGenericHttpMessageConverter
### writeInternal
write的核心处理方法writeInternal。

```javascript
protected void writeInternal(Object object, @Nullable Type type, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		MediaType contentType = outputMessage.getHeaders().getContentType();
		JsonEncoding encoding = getJsonEncoding(contentType);
		// com.fasterxml.jackson.core.JsonGenerator
		// JsonGenerator 中有输出流，这个输出流由 getBody 拿到（是 servletRespon 的输出流）
		JsonGenerator generator = this.objectMapper.getFactory().createGenerator(outputMessage.getBody(), encoding);
		try {
			// 输出开头
			writePrefix(generator, object);

			Object value = object;
			Class<?> serializationView = null;
			FilterProvider filters = null;
			JavaType javaType = null;

			if (object instanceof MappingJacksonValue) {
				MappingJacksonValue container = (MappingJacksonValue) object;
				value = container.getValue();
				serializationView = container.getSerializationView();
				filters = container.getFilters();
			}
			if (type != null && TypeUtils.isAssignable(type, value.getClass())) {
				javaType = getJavaType(type, null);
			}

			ObjectWriter objectWriter = (serializationView != null ?
					this.objectMapper.writerWithView(serializationView) : this.objectMapper.writer());
			if (filters != null) {
				objectWriter = objectWriter.with(filters);
			}
			if (javaType != null && javaType.isContainerType()) {
				objectWriter = objectWriter.forType(javaType);
			}
			SerializationConfig config = objectWriter.getConfig();
			if (contentType != null && contentType.isCompatibleWith(MediaType.TEXT_EVENT_STREAM) &&
					config.isEnabled(SerializationFeature.INDENT_OUTPUT)) {
				objectWriter = objectWriter.with(this.ssePrettyPrinter);
			}
		    // 输出 returnValue 到 generator
			objectWriter.writeValue(generator, value);
			
			// 输出结尾
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
	
	@Override
	public OutputStream getBody() throws IOException {
		this.bodyUsed = true;
		writeHeaders();
		return this.servletResponse.getOutputStream();
	}

	
	protected void writePrefix(JsonGenerator generator, Object object) throws IOException {
        if (this.jsonPrefix != null) {
            generator.writeRaw(this.jsonPrefix);
        }

        String jsonpFunction = object instanceof MappingJacksonValue ? ((MappingJacksonValue)object).getJsonpFunction() : null;
        if (jsonpFunction != null) {
            generator.writeRaw("/**/");
            generator.writeRaw(jsonpFunction + "(");
        }

    }

    protected void writeSuffix(JsonGenerator generator, Object object) throws IOException {
        String jsonpFunction = object instanceof MappingJacksonValue ? ((MappingJacksonValue)object).getJsonpFunction() : null;
        if (jsonpFunction != null) {
            generator.writeRaw(");");
        }

    }
```


## ObjectWriter
* com.fasterxml.jackson.databind.ObjectWriter
### writeValue
`objectWriter.writeValue(generator, value)` 方法中将value对象通过serialize序列化方法，将对象转为json字符串，然后设置到io流中。我们最后看看Jackson最终的序列化是怎么样的？

```javascript
@Override
    public final void serialize(Object bean, JsonGenerator gen, SerializerProvider provider)
        throws IOException
    {
        if (_objectIdWriter != null) {
            gen.setCurrentValue(bean); // [databind#631]
            _serializeWithObjectId(bean, gen, provider, true);
            return;
        }
        // 设置json的开始符号（"{"）
        gen.writeStartObject(bean);
        if (_propertyFilterId != null) {
           // 循环将对象设置为json字符串 serializeFieldsFiltered(bean, gen, provider);
        } else {
            serializeFields(bean, gen, provider);
        }
        // 设置json的结束符号（"}"）
        gen.writeEndObject();
    }
```

在serialize方法中通过JsonGenerator将要返回的对象转为json格式的字符串。
