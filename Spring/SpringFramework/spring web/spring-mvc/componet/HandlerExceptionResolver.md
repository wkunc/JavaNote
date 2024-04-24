# HandlerExceptionResolver

Spring MVC 中的异常处理.

```java
public interface HandlerExceptionResolver {

	@Nullable
	ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);

}
```

体现在DispastcherServlet的处理过程中就是

```java
try {
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
        processedRequest = checkMultipart(request);
        multipartRequestParsed = (processedRequest != request);

        // 1. 确定当前请求对应的 handler. 路由到具体的handler上
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
        }

        // 2. 确定当前 handler 对应的 HandlerAdpater.
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // 处理 Http 协议 last-modified header.
        String method = request.getMethod();
        boolean isGet = "GET".equals(method);
        if (isGet || "HEAD".equals(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                return;
            }
        }

        // 3. 调用 handler 的前置拦截器
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
        }

        // 4. 利用之前确定的 HandlerAdapter 调用 handler.
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        if (asyncManager.isConcurrentHandlingStarted()) {
            return;
        }

        // 5
        applyDefaultViewName(processedRequest, mv);
        // 6
        mappedHandler.applyPostHandle(processedRequest, response, mv);
    }
    catch (Exception ex) {
        dispatchException = ex;
    }
    catch (Throwable err) {
        // As of 4.3, we're processing Errors thrown from handler methods as well,
        // making them available for @ExceptionHandler methods and other scenarios.
        dispatchException = new NestedServletException("Handler dispatch failed", err);
    }
    // 可以看到上面try块代码中执行了如下步骤:
    // 1. 路由到handler
    // 2. 确定合适HandlerAdapter
    // 3. 前置拦截器
    // 4. 调用handler方法
    // 5. 解决视图名
    // 6. 后置拦截器
    // 结合catch块, 就发现我们捕捉了过程中的全部异常.
    // 从注释还可以看出, 在4.3之前只能处理 Exception, 之后也可以处理 Error
    // 最后就是这里处理结果的方法. 如果没有错, 会解析视图, 如果有异常会启动异常处理过程.
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
catch (Exception ex) {
    //...省略
}
catch (Throwable err) {
    //...省略
} finally {
    //...省略, 和异步相关?
}

/**
* 处理 handler 选择和 handler 的调用结果,
* 该结果可以是 ModelAndView 或要解析为 ModelAndView 的异常.
* 没有抛出异常就是 ModelAndView,
* 如果抛出了异常就是利用异常处理程序将 Exception 变成 ModelAndView
*/
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    boolean errorView = false;

    // 1. 在异常不为空的情况下, 就意味着处理过程中抛出了异常.
    // 调用异常处理过程 processHandlerException()
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            // 重点
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    // 2. 执行视图解析的过程, 在使用HttpMessageConverter的情况下不需要这一步
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isTraceEnabled()) {
            logger.trace("No view rendering, null ModelAndView returned.");
        }
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Concurrent handling started during a forward
        return;
    }

    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}

```

