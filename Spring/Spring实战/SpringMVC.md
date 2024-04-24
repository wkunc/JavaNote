# Spring MVC 流程

这里暂时不讲, 网上有很多类似的图解释

# Spring MVC 基于 java 配置

首先 Spring MVC 框架的入口是 DispatcherServlet ,我们需要在 web.xml 中配置它

```xml
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>*.form</url-pattern>
    </servlet-mapping>
```

在 Servlet3 规范中, 容器会在类路径中查找实现 javax.servlet.ServletContainerInitiallizer 的类, 如果发现了, 就会用它来配置
Servlet 容器

Spring 提供了这个接口的实现, 名为 SpringServletContainerInitalizer ,这个类会反过来查找 实现 WebApplicationInitializer
的类并将配置任务交给他们来完成
. Spring 3.2 引入了一个便利的 WebApplicationInitalizer 基础实现

```java
    public class AppWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
        //Spring 应用上下文位置类
        @Override
        protected Class<?>[] getRootConfigClasses() {
            return new Class[]{RootConfig.class};
        }
        //Servlet 应用上下文配置类
        //在里面配置 视图解析器, 静态资源处理,
        @Override
        protected Class<?>[] getServletConfigClasses() {
            return new Class[]{WebConfig.class};
        }
        //设置servlet映射路径
        @Override
        protected String[] getServletMappings() {
            return new String[]{"/"};
        }
    }
```

WebContext 配置

```java
    @Configuration
    @ComponentScan(basePackages = {"controller"})
    @EnableWebMvc
    public class WebConfig extends WebMvcConfigurerAdapter {
        //jsp 视图解析器
        @Bean
        public ViewResolver viewResolver(){
            InternalResourceViewResolver resolver = new InternalResourceViewResolver();
            resolver.setPrefix("/WEB-INF/views/");
            resolver.setSuffix(".jsp");
            resolver.setExposeContextBeansAsAttributes(true);
            return resolver;
        }
        //静态资源处理
        @Override
        public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
            configurer.enable();
        }
    }
```

RootContext配置

```java
    @Configuration
    @ComponentScan(basePackages = {"entity"})
    public class RootConfig {

    }
```

# Controller 细节

| 注解                   | 描述
|----------------------|------------
| @RequestMapping()    | 控制url映射关系
| @RequestParame()     | 获取请求参数
| @RequestHeader()     | 获取请求头
| @CookieValue()       | 获取cookie值
| @SessionAttributes() | 获取session值
| @ModelAttribute()    |

# Java Validaion API (JSR-303)

| 注解           | 描述
|--------------|------------------------------------------------------
| @AssertFalse | 所注解的元素必须是 **Boolean** 类型, 并且值为 False
| @AssertTrue  | 所注解的元素必须是 **Boolean** 类型, 并且值为 True
| @DecimalMax  | 所注解的元素必须是 **数字** , 并且它的值要小于或等于给定的 BigDecimalString 值
| @DecimalMin  | 所注解的元素必须是 **数字** , 并且它的值要大于或等于给定的 BigDecimalString 值
| @Digits(数字)  | 所注解的元素必须是 数字 , 并且它的值必须有指定的位数
| @Future      | 所注解的元素必须是一个 **将来的日期**
| @Past        | 所注解的元素必须是一个 **已经过去的日期**
| @Max         | 所注解的元素必须是 **数字** , 并且它的值要小于或等于给定的值
| @Min         | 所注解的元素必须是 **数字** , 并且它的值要大于或等于给定的值
| @NotNull     | 所注解的元素值 不能为 null
| @Null        | 所注解的元素值 必须为 null
| @Pattern     | 所注解的元素值必须匹配所给定的正则表达式
| @Size        | 所注解的元素值必须是 String, 集合或数组, 并且它的长度要符合给定的范围

# Spring MVC 的高级技术

## 处理 multipart 形式的数据

配置 mulitpart 解析器

DisparcherServlet 并没有实现任何解析 multipart 请求疏解的功能

它将任务委托给了 Spring 中 MultipartResolver 策略接口的实现, 通过这个实现类来解析 multipart 请求中的内容.

从 Spring 3.1 开始, Spirng 内置了两个 MultipartResolver 的实现供我们选择:

* CommonsMultipartResolver : 使用 Jakarta Commons FileUpload 解析 multipart 请求
* StandardServletMultipartResolver 依赖于 Spring 3.0 对 multipart 请求的支持

一般来说 第二种 更好 ,它使用 Servlet 所提供的功能支持, 并不需要依赖任何其他项目

***
使用 Servlet 3.0 解析 multipart 请求

兼容 Servlet 3.0 的 StandardServletMultipartResovler 没有构造参数, 也没有要设置的属性, 所以在 Spring 应用上下文中 声明
它会非常简单

```java
    //multipart 请求处理器
    @Bean
    public MultipartResolver multipartResolver(){
        return new StandardServletMultipartResolver();
    }
```

但是 我们如何控制 这个 multipart 请求处理器的行为呢? 如文件上传位置, 限制文件大小等.

我们必须在 web.xml 或 Servlet 初始化类中, 将 multipart 配置的细节 作为 DispatcherServlet 配置的一部分.

```java
public class AppWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
    //配置 multipart 处理器的 参数
    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
        registration.setMultipartConfig(new MultipartConfigElement("/tmp/vr/uploads"));
    }
}
```

## 编写控制器方法

```java
	@RequestMapping("/testMultipart")
	public String fileUpload(@RequestPart("picture") byte[] pricture){
        //文件处理...
	    return SUCCESS;
    }
```

这是最简单的 multipart 请求处理器, 用 @RequestPart 注解获取文件,用 byte[] 来接收二进制部分.

使用上传文件的原始 byte 比较简单但是功能有限. 因此 Spring 还提供了 MultipartFile 接口, 它为处理 multipart
数据提供了内容更为丰富的对象, 如下展示了 MultipartFile 接口的概况

```java
public interface MultipartFile extends InputStreamSource {
    String getName();
    String getOriginalFilename();
    String getContentType();
    boolean isEmpty();
    long getSize();
    byte[] getBytes() throws IOException;
    InputStream getInputStream() throws IoExcepation;
    void transferTo(File dest) throws IOException, IlleaglStateException;
}
```

# 处理异常

不管出现什么情况, servlet 的请求输出都是一个 Servlet 响应. 如果在请求处理的时候, 出现了异常, 那它的输出依然还是 Servlet
相应, 异常必须要以 某种方式转换为响应

Spring 提供了多种方式将异常转换为响应:

* 特定的 Spring 异常会将自动映射为指定的 **HTTP 状态码**
* 异常上可以添加 **@ResponseStatus** 注解, 从而将其映射为某一个 HTTP 状态码
* 在方法上可以添加 **@ExceptionHandler** 注解, 使其来处理异常

## 使用 @ResponseStatus 注解

```java
@ResponseStatus(code = HttpStatus.NOT_FOUND, reason = "User Not Found")
public class UserNotFoundException extends RuntimeException {

}
```

## 使用 @ExceptionHandler 注解

```java
    @ExceptionHandler(DuplicateSpittleException.class)
    public String handlerDuplicateApp(){
        return "error/duplicate";
    }
```

## 为控制器添加通知

如果多个控制器中都会 抛出某个特定异常 那么每个控制器中都会需要重复编写 一样的 @ExceptionHandler 方法, 为了解决这类问题
Spirng 3.2 中引进了一个新的解决方案: 控制器通知(Controller aadvice)

控制器通知是任意带有 **@ControllerAdvice** 注解的类, 这个类会包含一个或多个如下类型的方法

* @ExceptionHandler 注解标注的方法
* @InitBinder 注解标注的方法
* @ModelAttribute 注解标注的方法

在带有 **@ControllerAdvice** 注解的类中, 以上所述的所有方法会运用到整个应用程序中 **所有的** 控制器中带 @RequestMapping
注解的方法上

```java
    @ControllerAdvice
    public class AppWideExceptionHandler {
        @ExceptionHandler(UserNotFoundException.class)
        public String handlerDuplicateApp(){
            return "error/duplicate";
        }
        //其他类似的方法
    }
```

这样我们就不用在每一个控制器中编写 那些重复的方法

@ControllerAdvice 注解已经使用了 @Component 注解, 所以 Spring 也会通过组件扫描获取到被标注的类
