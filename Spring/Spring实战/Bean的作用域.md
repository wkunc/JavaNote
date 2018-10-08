# Bean 的作用域
在默认情况下, Spring 应用上下文中的所有 bean 都是作为以单例(singletion) 的形式创建的

## Spring中的作用域
* 单例(singletion) 在整个应用中, 只创建 bean 的一个实例
* 原型(Prototype) 每次注入, 或者通过 Spring 应用上下文获取的时候, 都会创建一个bean 实例
* 会话(Session) 在 Web 应用中, 为每一会话创建一个 bean 实例
* 请求(Request) 在 Web 应用中, 为每一次的请求创建一个 bean 实例

## 解决方式 @Scope 注解
原型的声明方式
```java
    //@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    @Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    @Bean
    public User user(){
        return new User();
    }
```
```java
    //@Scope("prototype")
    @Scope(scopeName = "prototype")
    @Bean
    public User user(){
        return new User();
    }
```

### session 和 request 作用域比较特殊

下次写

# 运行时值注入

就是避免硬编码, 用.properties, .xml等属性文件解决硬编码

支持两种方式 获取属性文件中的值 **属性占位符** 和 **SpEL**(Spring 表达式语言)

## 第一步 引入属性文件

### @PropertySource

在基于java 的配置中可以用 @PropertySource 来引入属性文件
```java
    @PropertySource("classpath:/resources/app.properties")
    @Configuration
    public class RootConfig {
        //..其他配置
    }
```

## 第二步 通过 Spring Environment 使用属性值
```java
    @PropertySource("classpath:/resources/app.properties")
    @Configuration
    public class RootConfig {
        //..其他配置
        @Autowired
        Environment env;

        @Bean
        public User user(){
            User user = new User();
            //通过 spring Environment 环境提供的方法获取属性文件内容
            user.setName(env.getPropertype("name"));
            return user;
        }
    }
```

> ### 深入了解 Spring 的 Environment
>
> getProperty() 方法有四种重载形式
>
>* String getPropery(String key)
>* String getPropery(String key, String defaultValue)
>* T getPropery(String key, Class<T> type)
>* T getPropery(String key, Class<T> type, T defaultValue)

下次再写

## 解析属性占位符

直接从 Environment 中检索属性是非常方便的， 尤其是用 java 配置中

但是 Spring 也提供了通过占位符配装属性的方法， 这些占位符的值会来自一个属性源

在 XML 配置中占位符的形式是 __${属性名}__ 但是在 java 配置中这并不适用

为了使用占位符 Spring 引入了 @Value 注解

### @Value 注解的使用
@Value 注解有些类似 @Autowired 注解, 只不过它注入的是 属性文件中的值, 而不是 Spring 中 bean

当然 为了使用 @Value 注解 我们还要配置一个 Spring 提供的 bean
PropertyPlaceholderConfigurer 或者 PropertySourcesPlaceholderConfiguer. 从 Spring 3.1 开始推荐使用后者

```java
    public class User {
        private String name;

        public User(){ }

        public User(@Value(${"name"}) String name) {
            this.name = name;
        }
        //其他部分...
```
显式的声明 和 自动扫描时 @Value 注解标注的位置不一样

```java
    @Bean
    public User user(@Value(${"name"})String name){
        return new User(name);
    }
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer(){
        return new PropertySourcesPlaceholderConfigurer();
    }
```

## 使用 Spring 表达式语言进行配装

SpEL 是 **属性占位符** 的升级版, 它能做到许多属性占位符做不到的事

## SpEL 特性列表
> * 使用 bean 的ID 来引用 bean
> * 调用方法和访问对象的属性
> * 对值进行算术, 关系和逻辑运算
> * 正则表达式匹配
> * 集合操作
> 
