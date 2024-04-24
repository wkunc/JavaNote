# 基于java的配置

基于Java的配置首先有一个用 @Configuratino 被标明为配置类的java类

```java
@Configuration
public class RootConfig {

}
```

## 显式配置

然后在配置类中写返回对象的方法, 并用 @Bean标志 告知spring的 IOC 容器这是一个 交给你管理的 Bean

默认情况下 bean 的 id 和被标注的方法名相同

也可以用@Bean(name="id")来指定一个不同的名字

```java
@Configuration
public class RootConfig {
    @Bean
    public User getUser(){
        return new User();
    }
}
```

## 显式的注入

### 构造器注入

这里要求 Order 有一个带参构造器

```java
    @Bean
    public User user(){
        return new User();
    }
    @Bean
    public Order order(){
        return new Order(user());
    }
```

这种写法看上去在生成 Order 对象时要 调用 user()方法, 实际上并不会, 因为 user() 已经被@Bean标注了每次调用 user() 方法都会被
spring 拦截, 然后返回 该方法返回的 bean.

如果觉得这种写法容易误解下面还有一种写法

```java
    @Bean
    public User user(){
        return new User();
    }
    @Bean
    public Order order(User user){
        return new Order(user);
    }
```

当 spring 调用 order() 方法来创建一个 Order 对象时, 它会自动装配一个 User 到这个方法中.

这种引用其他的 bean 通常时最佳选择, 因为它不会要求 order 和 user 声明在同一个配置类中, 甚至没有要求 User 必须在
javaConfig 中声明, 实际上 User 还可以是 自动扫描找到的(没有显式声明).

总结: 不管是什么方式给spring管理的bean 都能用第二种方式装配

注意: 这两种方式都使用了构造器注入, 这要求Order 必须有一个对应的 带参构造器

### Setter 方法注入

这种要求对应的 bean 含有 setter 方法

```java
    @Bean
    public User user(){
        return new User();
    }
    @Bean
    public Order order(){
        Order order = new Order();
        order.setUser(user());
        return order;
    }
```

和构造器注入一样这里也有两种方式, 这两种方式的区别和构造器注入也是一致的

```java
    @Bean
    public User user(){
        return new User();
    }
    @Bean
    public Order order(User user){
        Order order = new Order();
        order.setUser(user);
        return order;
    }
```

注意: 第二种方式也不是完美的, 因为这里希望 spring 自动的注入我们想要的对象 如果我们想要的对象 User 在 spring 中有多个会引起歧义,
它不知道该给这个方法注入哪一个 User ,这会引发一个异常NoUniqueBeanDefinitionException (bean 不唯一)

经检验 spring 不会在任意情况都报 NoUniqueBeanDefinitionException 异常如:

```java
    @Bean
    public User user(){
        return new User("bill");
    }
    @Bean
    public User getUser(){
        return new User("bob");
    }
    @Bean
    public Order order(User user){
        Order order = new Order();
        //此时的 User 是 bill
        order.setUser(user);
        return order;
    }
````

虽然这时我们在 spring 中配置了 两个User 类型的对象,但是 spring 通过 变量名 user 判断 为第一个User对象然后将其 注入 Order
类中. 如何更好的解决 bean 不唯一将在高级装配中讲到

### 总结

第二种方式中 spring 会先通过类型判断, 如果有相同类型的在通过 变量名判断

### @Bean 属性表

| attribute     | Description
|---------------|---------------------------------------
| name          | String[]类型 自定义的给 bean 起id
| value         | 和 name 属性一样, 这两个互为别名
| initMethod    | String类型 初始化方法名 default = ""
| destroyMethod | String类型 销毁方法名 defualt = "(inferred)"

---------------------

## 隐式的配置(自动扫描)

自动扫描可以帮助我们减少许多重复性的配置

在配置类上用 @ComponentScan 开启 spring 的自动扫描功能

```java
@Configuration
@ComponentScan
public class RootConfig {

}
```

这样 spring 就会自动扫描 配置类所在的包以及所有的子包中的 被标明Bean如:

```java
@Component
public class User {
    private String name;
    //..构造器, getter setter
}
```

@Component, @Controller, @Service, @Repository 都可以标注要被 自动注册的类, 而且在源码中 它们基本没有区别, @Controller,
@Service, @Repository都用了 @Component标注了

@ComponentScan可以用 value 属性指定扫描的包

```java
@Configuration
@ComponentScan("entity")
//@ComponentScan(value = {"entity"})和上面作用完全一致
public class RootConfig {
}
```

## 隐式的注入(自动装配)

@Autowired可以用在下面几个方面

* 构造器
* 属性
* setter方法

强依赖建议通过构造器注入, 并且加入断言检查

通过属性注入 (Spring 团队不推荐使用)

```java
@Component
public class Order {
    @Autowired
    private User user;
    //.其他内容
}
```

通过构造器的自动装配

```java
@Component
public class Order {
    private User user;

    @Autowired
    public Order(User user){
        this.user = user;
    }
}
```

通过setter方法的自动装配

```java
@Component
public class Order {
    private User user;

    @Autowired
    public void setUser(User user) {
        this.user = user;
    }
}
```

# 导入和混合配置

## 在 javaConfig 中引用 其他的 javaConfig

用 @Import 标记

```java
@Import(DataConfig.class)
@Configuration
@ComponentScan("entity")
public class RootConfig {

}
```

也可以一次加载多个 @Import({DataConfig.class,WebConfig.class})

## 在 javaConfig 中引用 XML 配置

用 @ImportResource 标记加载

```java
@ImportResource("Bean.xml")
@Configuration
@ComponentScan("entity")
public class RootConfig {

}
```

也可以一次加载多个xml
@ImportResource({"Beans.xml","otherBeans.xml"})
