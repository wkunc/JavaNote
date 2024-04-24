# Annotated Controllers

Spring MVC 提供了一个基于注解的编程模型, 其中@Controller和@RestController组件
使用注解来表达请求映射, 请求输入, 异常处理等,

# Handler Methods

Method Arguments

## @RequestParam

使用@RequestParam注解将 Servlet 请求参数(即查询参数和表单数据)绑定到控制器中的方法参数.

默认情况下, 任何简单值类型并且不被任何其他参数解析器解析, 都被视为使用 @RequestParam.
(简单值类型的判断由 BeanUtils.isSimpleProperty()确定.)

## @ModelAttribute

使用@ModelAttribute注解来从模型访问属性, 或者如果不存在则将其实例化.
Model属性还覆盖了名称与字段匹配的Http Servlet请求参数值. 这被称为数据绑定.

使用位置如下:

1. 在@RequestMapping方法中的方法参数, 用于从模型创建或访问Object并通过WebDataBinder将其绑定到请求.
2. 作为@Controller或@ControllerAdvice类中的方法级注释, 有助于在任何@RequestMapping方法调用之前初始化模型.
3. 在@RequestMapping方法上标记其返回值是一个模型属性.
   本节只讲第一种情况.

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { }
```

这个 Pet 实例会从以下方面获得:

1. 如果存在已使用的Model, 从模型中获取
2. 从Session对象中获取.
3. 使用 Converter 对象通过 URI path variable获取.
4. 从默认构造器调用, new 一个实例.
5. 调用具有与 Servlet 请求参数匹配的构造器. 参数名通过 @ConstructorProperties 或 class 中保留的参数名确定.

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```

这个@ModelAttribute和路径变量 account 匹配. 它会通过一个已经注册的 Converter\<String, Account>来加载对象.

以上步骤获得到对象实例后, 将执行数据绑定的步骤.
WebDataBinder类将Servlet请求参数名和字段名进行匹配, 在必要时使用类型转换后填充字段.
数据绑定可能导致错误, 默认情况下, 会引发BindException.

## @SessionAttributes

这是一个类级别的注解, 还有一个和它很像的是方法参数的注解@SessionAttribute.

# Model

@ModelAttribute 方法具有灵活的方法签名. 它们支持许多与@RequestMapping方法相同的参数,
除了它自己和任何有关request body有关的内容, 不能做方法参数.

```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
// 通常是不必要的.
// 当返回值不是一个 Model 的时候, 会自动将返回值添加到当前Model, 所以不必要.
// 不过可以通过这个注解设置name, 这样就避免了默认命名
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```
