# HandlerMethodReturnValueHandler

```java
public interface HandlerMethodReturnValueHandler {
    
    boolean supportsReturnType(MethodParameter returnType);

    /*
    * 处理给定的返回值, 添加属性到模型中并且设置 View 或者设置
    * ModelAndViewContainer.setRequestHandled(true) 表示响应已直接处理
    */
    void handleReturnValue(Object returnValue, MethodParameter returnType,
                ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throw Exception;

}
```
