# Bean 创建过程

1. prepareMethodOverrides() (对应方法注入的功能)

这个方法是为了Spring为其创建一个代理提供机会, 如果真的创建了一个proxy, 那么还需要为这个代理调用 BeanProcessor 的AfterProcessor(后置处理)
2. resolveBeforeInstantiation() InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation(Class, BeanName)

3. MergedBeanDefinitionPostProcessor 的 postProcessMergedBeanDefinition()

4. initalizeBean() 执行一系列的Bean初始化方法
> 具体顺序如下: 
> 1. invokAwareMethods()负责调用和IOC密切相关的Aware如BeanFactoryAware.
> 2. applyBeanPostProcessorsBeforeInitialization() 调用BeanPostProcessor 的Bean初始化前置方法 
> 3. invokeInitMethods() 调用 InitializaingBean 接口定义的初始化方法或调用用户自定义的init-method方法

5. 

