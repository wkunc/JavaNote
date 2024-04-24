# HandlerAdapter

这个接口为了可以扩展所以 handler 对象是一个 Object.
这样就可以支持不同的 handler 类型了. 无论是注解驱动的handler 还是其他框架的处理对象.
这个接口对象可以实现 Ordered 接口允许指定优先级.
没有实现Ordered接口的 Adapter 对象会拥有低优先级.

```java
public interface HandlerAdapter {
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest request, HttpServletResponse, Object handler) throws Exception;
    long getLastModified(HttpServletRequest request, Object  handler);
}
```

# HttpRequestHandlerAdapter

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {
    public boolean supports(Object handler) {
        return (handler instanceof HttpRequestHandler);
    }
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse )
            throws Exception {

        ((HttpRequestHandler) handler).handleRequest(request, response);
        retrun null;

    }

    public long getLastModified(HttpServletRequest request, HttpServletResponse response) {
        if (handler instanceof LastModified) {
            return ((LastModified) handler).getLastModified(request);
        }
        retrun -1;
    }
}
```

# RequestMappingHandlerAdapter

```java
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
    private int order = Ordered.LOWEST_PRECEDENCE;

    public AbstractHandlerMethodAdapter() {
        // 默认情况下不限制 Http Method
        super(false);
    }
}
```
