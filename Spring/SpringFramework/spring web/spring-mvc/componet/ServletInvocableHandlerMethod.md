# InvocableHandlerMethod

![](HandlerMethod.png)
HandlerMethod 的扩展(我们想调用HandlerMethod, 必须自己提供其包装方法的参数).
而这个扩展类就是提供了一个自动从 Request 请求中自己解决参数的行为.
既然方法参数都已经解决了, 那么其包装的Method自然可以进行调用了.

它通过HandlerMethodArgumentResolver list, 然后调用方法返回对象.

```java
public class InvocableHandlerMethod extends HandlerMethod {
	private static final Object[] EMPTY_ARGS = new Object[0];

    // 非常重要的 DataBinderFactory. 主要是传递给 HandlerMethodArgumentResolver ,用来绑定数据.
    // 不是所有的接口实现都会用到这个 DataBinderFactory.
	@Nullable
	private WebDataBinderFactory dataBinderFactory;

    // 能够自己解决方法参数的核心.
	private HandlerMethodArgumentResolverComposite resolvers = new HandlerMethodArgumentResolverComposite();

    // 帮助获取参数名的工具, 要知道Java在编译的时候默认会把参数名抹去.
    // 虽然我也不是很懂它是怎么做的, 但是它的功能就是获取参数名
	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    // 构造器没什么好说的, 就是把父类HandlerMethod的构造器抄了一遍
	public InvocableHandlerMethod(HandlerMethod handlerMethod) {
		super(handlerMethod);
	}
	public InvocableHandlerMethod(Object bean, Method method) {
		super(bean, method);
	}
	public InvocableHandlerMethod(Object bean, String methodName, Class<?>... parameterTypes)
			throws NoSuchMethodException {

		super(bean, methodName, parameterTypes);
	}
}
```

提供了 invokeForRequest(NativeWebRequest, ModelAndViewContainer, Object...)方法.

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }
    return doInvoke(args);
}

protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    // 获取内部的Method的所有参数.
    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    // 准备一个数组做容器, 因为大小是固定的, 所以不需要使用ArrayList等集合类.
    Object[] args = new Object[parameters.length];
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
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        }
        catch (Exception ex) {
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

# ServletInvocableHandlerMethod

如果说上面的子类提供了参数解析的功能,
那么这个类就提供了返回值解析的功能.
利用内部的 HandlerMethodReturnValueHandler list,

提供了 invokeAndHandle(ServletWebRequest, ModelAndViewContainer, Object...)方法.

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

