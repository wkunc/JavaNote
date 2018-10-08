# 配置 profile
主要是标明每个 bean 所属的 profile

## @Profile
这个注解可以用在配置类的上面, 表示这个配置类中的所有 bean 都属于某一个 profile

也可以用在 @Bean 方法上面,标注某一个 bean 是 哪个 profile的

在 Spring4 之后这个注解是基于下面那个注解实现的 @Conditioinal

* 用在配置类上
```java
@Profile("QA")
@Configuration
@ComponentScan("entity")
public class RootConfig {
    @Bean(destroyMethod ="close")
    public DataSource dataSource(){
        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/wkc");
        dataSource.setDriverClassName("com.mysql.Driver");
        dataSource.setJmxName("root");
        dataSource.setPassword("123456");
        return dataSource;
    }
}
```
* 用在 @Bean 方法上
```java
@Configuration
@ComponentScan("entity")
public class RootConfig {
    @Bean(destroyMethod ="close")
    @Profile("QA")
    public DataSource dataSource(){
        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/wkc");
        dataSource.setDriverClassName("com.mysql.Driver");
        dataSource.setJmxName("root");
        dataSource.setPassword("123456");
        return dataSource;
    }
}
```

## 激活 profile

### 关键属性 active 和 default
Spring 在确定哪个 profile 处于激活状态时, 需要依赖两个独立属性: spring.profiles.active 和 spring.profiles.default. 

如果设置了spring.profiles.active 属性的话, 那么它的值会用来确定哪个 profile 时激活的. 

但如果没有设置 spring.profiles.active属性的话, 那Spring 将会查找spring.profiles.default 的值. 

如果都没有设置就没有激活的 profile ,因此只会创建那些没有定义在 profile 中 bean

### 设置方式
有多种方式来设置这两个属性:
* 作为 DispathcherServlet 的初始化参数
* 作为 Web 应用的上下文参数
* 作为 JNDI 条目
* 作为环境变量
* 作为 JVM 的系统属性
* 在集成测试类上, 使用 @ActiveProfiles 注解设置

重点 : DispatcherServlet, 和 集成测试

# 条件化的 bean
是希望 只有在应用的类路径下包含特定的库时才创建某个 bean

希望某个 Bean 只有当另外某个特定的 bean 也声明之后才创建

我们还可能要求只有在某个环境变量设置后, 才创建某个 bean

在 Spring4 之前很难实现这种带级别的条件化配置, Spring4 引入了一个新的注解 @Conditional ,它可以用到 带有@Bean 注解的方法上. 如果给定的条件计算结果为 true 就会创建这个 bean 否则的话, 这个 bean 会被忽略

```java
    @Conditional(MagicExistsCondition.class)
    @Bean
    public User user(){
        return new User();
    }
```
这个注解必须传入一个实现了 Condition 接口的类
```java
public interface Condition {
    boolean matches(ConditionContext var1, AnnotatedTypeMetadata var2);
}
```
