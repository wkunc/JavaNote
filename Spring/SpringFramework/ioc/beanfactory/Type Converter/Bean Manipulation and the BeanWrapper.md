# Bean Mainpulation and the BeanWrapper

org.springframework.beans 包中一个非常重要的类是 `BeanWrapper`以及其实现类
`BeanWrapperImpl`.

`BeanWrapper`提供了
1. set and get property value (单个或者多个)
2. get property descriptors
3. query properties to determine if they are readable or writeable
4. support nested properties. 运行无限深度的子属性设置
5. support standard JavaBeans `PropertyChangeListeners` and `VetoableChangeListeners`



## Built-int propertyEditor Implementations

Spring 使用 `PropertyEditor` 抽象来实现 Object 和 String 之间的转换.

> 因为 xml 配置中都是 String, 所以需要提供一个将 String 转换为对应的类型的工具
> 比如将 xml 中的 日期字符串 转换成 Date 类型
> 而且 web 环境下也有这种需求, 毕竟http请求里的内容本质都是 String.
> 而我们通常需要转换, Spring 帮开发者做这种事, 减少了重复劳动


## Spring Type Conversion
在Spring 3时, 引入了 core.convert 包.  提供了 `通用` 的类型转换系统.

该系统定义了一个用户实现类型转换逻辑的 SPI 
和一个用于在运行时执行类型转换的API.
在 Spring 容器中, 可以使用此系统作为 `PropertyEditor` 实现的替代方法,
以将外部化的 bean 属性字符串转换为所需要的属性类型.

也可用于开发者的应用程序中任何需要类型转换的地方.

> 主要是 PropertyEditor 系统, 没办法让开发者使用.
> 而新的 ConverterService 可以方便的给开发者使用.
>
> 根据观察, 默认情况下 Spring 没有使用新的
> 在web环境中开启 @EnableWebMvc 时, 会配置一个 ConverterService 来取代.
