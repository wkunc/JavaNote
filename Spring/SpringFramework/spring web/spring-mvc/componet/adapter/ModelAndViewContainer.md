# ModelAndViewContainer

在调用控制器方法的过程中记录由 HandlerMethodArgumentResolvers 和
HandlerMethodReturnValueHandlers 做出的模型和视图相关的决策.

简单的说就是保存视图和模型的容器, 还包括了相关的选项.
如: setRequestHandled 可以指示已经处理请求不需要查看相关解析.

在实例化时自动创建默认模型,
可以通过 setRedirectModel 提供备用模型实例以用于重定向场景.

```java
public class ModelAndViewContainer {

    //
	private boolean ignoreDefaultModelOnRedirect = false;

    // View 对象, 可能是 viewName, 也可以是View接口实现类
	@Nullable
	private Object view;

    // 模型数据,
	private final ModelMap defaultModel = new BindingAwareModelMap();

    // 重定向模型数据, 故意不初始化的.
    // 基本上只有在处理 "redirect: /sprng" 类型的字符串返回值
    // 和显式的RedirectAttribute返回值是会设置.
	@Nullable
	private ModelMap redirectModel;

	private boolean redirectModelScenario = false;

	@Nullable
	private HttpStatus status;

	private final Set<String> noBinding = new HashSet<>(4);

	private final Set<String> bindingDisabled = new HashSet<>(4);

	private final SessionStatus sessionStatus = new SimpleSessionStatus();

    // 标志当前request是否已经处理完成
	private boolean requestHandled = false;

    // 没有显式构造器.
}
```

这个Model和View的容器中有两个 Model, 和一个View.
两个Model看起来有点奇怪, 明明只需要一个容器就可以了.
两个Model是为了支持重定向,

重点方法

```java
public ModelMap getModel() {
    if (useDefaultModel()) {
        return this.defaultModel;
    }
    else {
        if (this.redirectModel == null) {
            this.redirectModel = new ModelMap();
        }
        return this.redirectModel;
    }
}
// 判断是选择使用哪一个Model(是default Model 还是 redirect model)
private boolean userDefaultModel() {
    return (!this.redirectModelScenario || (this.redirectModel==null&&!this.ignoreDefaultModelOnRedirect))
}
```

虽然提供了直接gett default Model 的方法, 但是不建议在普通情况下使用.
