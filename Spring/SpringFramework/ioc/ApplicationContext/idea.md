# 关于 @Configuration 类中调用@Bean方法
Spring实战这本书提到过, 在配置类中的方法中调用@Bean方法, 不会真正执行方法,
而是被变成从 Spring 中获取对应Bean的引用.
```java
@Configuration
class RootConfig {
    @Bean
    public DataSource dataSource() {
        return new SimpleDataSource("root"....);
    }

    @Bean
    public JapEntityFactory entityFactory() {
        ....
        return new EntityFactory(dataSource());
    }
}
```
> 上面在创建 JPA 实体工厂时需要配置数据源
> 在@Bean方法中调用了另一个@Bean方法
> 按正常的Java流程来说, 这会调用dataSource()方法从而获取一个DataSource
> 但是这个方法已经注册为了Bean. 
> 也就是说如果没有Spring的这个功能, 上面的代码会导致我们有两个独立的 DataSource,
> 在使用JPA时完全没有使用注册在IOC容器中的那个, 那么我们通过的IOC容器解耦也就变成了笑话.
> 但是实际上 Spring 会拦截这个方法调用, 使这个方法调用变成从 IOC 容器中获取对应的引用.
> 这样就不会导致重复Bean了, 确保了每个Bean都被容器管理, 实现解耦.

上面讲了这么多Spring 拦截@Bean方法功能的作用, 那么它是怎么实现的呢?
通过Trace级别的日志, 我们可以看到Spring为每个配置类通过生成GGLIB子类来代理.
从而增强了配置类拦截了@Bean方法调用.

具体可以看 ConfigurationClassEnhancer (它负责生成代理类)
而我们熟悉的 ConfigurationClassPostProcessor 负责在Context启动前找出配置类定义,
并且使用上面的类为其生成增强类.
