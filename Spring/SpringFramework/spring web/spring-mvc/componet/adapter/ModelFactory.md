# ModelFactory

用于再调用控制器方法前初始化 Model(就是调用@ModelAttribute方法和解析Session值),
和再方法调用之后进行 update Model(还不知道更新是什么意思).

所以这个类的public方法是:

1. initModel()
2. updateModel()

```java
public final class ModelFactory {

	private static final Log logger = LogFactory.getLog(ModelFactory.class);

    // 需要调用的 @ModelAttribute 方法集.
	private final List<ModelMethod> modelMethods = new ArrayList<>();

	private final WebDataBinderFactory dataBinderFactory;

    // 负责@SesssionAttributes信息
	private final SessionAttributesHandler sessionAttributesHandler;

	public ModelFactory(@Nullable List<InvocableHandlerMethod> handlerMethods,
			WebDataBinderFactory binderFactory, SessionAttributesHandler attributeHandler) {

		if (handlerMethods != null) {
			for (InvocableHandlerMethod handlerMethod : handlerMethods) {
				this.modelMethods.add(new ModelMethod(handlerMethod));
			}
		}
		this.dataBinderFactory = binderFactory;
		this.sessionAttributesHandler = attributeHandler;
	}
}
```

根据构造器我们知道要创建一个 ModelFactory,
需要准备一个 SessionAttributesHandler
和需要调用的 @ModelAttribute 方法集
以及一个 DataBinderFactory

RequestMappingHandlerAdapter 中的创建过程.

```java
// 传入的 Factory 是之前的方法创建好的.
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
    // 调用的 getSeesionAttributesHandler() 方法, 获取SeesionAttributesHandler
    SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);

    // 下面和创建DataBinderFactory的流程一样, 只是查找的注解方法不同
    Class<?> handlerType = handlerMethod.getBeanType();
    Set<Method> methods = this.modelAttributeCache.get(handlerType);
    if (methods == null) {
        methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
        this.modelAttributeCache.put(handlerType, methods);
    }
    List<InvocableHandlerMethod> attrMethods = new ArrayList<>();
    // Global methods first
    this.modelAttributeAdviceCache.forEach((clazz, methodSet) -> {
        if (clazz.isApplicableToBeanType(handlerType)) {
            Object bean = clazz.resolveBean();
            for (Method method : methodSet) {
                attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
            }
        }
    });
    for (Method method : methods) {
        Object bean = handlerMethod.getBean();
        attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
    }
    // 最后调用构造器, 创建完成
    return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}
```

# 模型数据的初始化

```java
public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod)
        throws Exception {

    // 获取Session数据
    Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
    // 用Session数据填充Model
    container.mergeAttributes(sessionAttributes);
    // 调用@ModelAttribute方法填充Model
    invokeModelAttributeMethods(request, container);

    for (String name : findSessionAttributeArguments(handlerMethod)) {
        if (!container.containsAttribute(name)) {
            Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
            if (value == null) {
                throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
            }
            container.addAttribute(name, value);
        }
    }
}
```
