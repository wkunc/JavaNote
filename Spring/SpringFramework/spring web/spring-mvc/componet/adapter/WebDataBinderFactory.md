# WebDataBinderFactory
作为工厂主要功能就是可以创建 WebDataBinder,
而InitBinderDataBinderFactory 提供了一个用方法修改创建出来的 DataBinder 的机会.


主要在 RequestMappingHandlerAdapter.getDataBinderFactory方法调用

继承结构...
```java
public class InitBinderDataBinderFactory extends DefaultDataBinderFactory {

    // 会用在创建 DataBinder 的时候.
	private final List<InvocableHandlerMethod> binderMethods;

	public InitBinderDataBinderFactory(@Nullable List<InvocableHandlerMethod> binderMethods,
			@Nullable WebBindingInitializer initializer) {

		super(initializer);
		this.binderMethods = (binderMethods != null ? binderMethods : Collections.emptyList());
	}
}
```

主要是这一层, 要求在初始化中传入可调用的 @InitBinder 方法的 HandlerMethod对象集合.

接下来看 RequestMappingHandlerAdapter 是如何创建这个 DataBinderFactory 的

```java
// 这里传入的 HandlerMethod 是@RequestMapping方法.
private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
    Class<?> handlerType = handlerMethod.getBeanType();
    // 从缓存中获取, 避免每次重复查找影响性能.
    Set<Method> methods = this.initBinderCache.get(handlerType);
    // 如果是第一次调用的 控制器 时没有缓存, 需要查找.
    if (methods == null) {
        // 查找 HandlerMethod 属于的控制器类中的 @InitBinder 方法. 
        // 并加入缓存, 以后调用这个控制器的任何 @RequestMapping 方法就不需要再查找了.
        methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
        this.initBinderCache.put(handlerType, methods);
    }
    List<InvocableHandlerMethod> initBinderMethods = new ArrayList<>();

    // 这里是查找全局的 @InitBinder 方法, 就是 @ControllerAdvice 类中的. 
    // 并为这些全局的 @InitBinder 方法调用 createInitBinderMethod() 方法.
    this.initBinderAdviceCache.forEach((clazz, methodSet) -> {
        if (clazz.isApplicableToBeanType(handlerType)) {
            Object bean = clazz.resolveBean();
            for (Method method : methodSet) {
                initBinderMethods.add(createInitBinderMethod(bean, method));
            }
        }
    });
    // 然后为每个局部的 @InitBinder 方法调用 createInitBinderMethod() 方法.
    for (Method method : methods) {
        Object bean = handlerMethod.getBean();
        initBinderMethods.add(createInitBinderMethod(bean, method));
    }
    // 最后调用 DataBinderFactory 的构造器传入准备好的HandlerMethod集合.
    return createDataBinderFactory(initBinderMethods);
}
```

