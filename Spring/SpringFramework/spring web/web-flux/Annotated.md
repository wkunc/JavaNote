# Annotated Controllers

# Request Mapping

## URI Patterns
你可以使用 glob 模式和通配符来映射request. 
(glob 模式指 shell 所使用的简化的正则表达式)

? 匹配一个字符
\* 匹配0个或多个字符
** 匹配0个或多个段

还可以通过 @PathVarible 声明URI变量并访问它们的值, 这就是路径变量.

```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
```

# Handler Methods

Method 参数注解

## @RequestParam
使用@RequestParam注解绑定查询参数到方法参数中.

Servlet API 中的"request parame" 概念是
查询参数, 表单数据, Multparts 的集合体.

而在WebFlux 中, 它们是通过 ServerWebExchange 分开访问的.
所以 @RequestParam 仅仅绑定查询参数.
你可以使用数据绑定来将 查询参数, 表单数据, multiparts 绑定到 command object.
```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }
    // ...
}
```

> 注意 : 使用@RequestParam 是可选的, 意思就是写不写随你. 
> 默认情况下任何属于简单值类型的参数
> 并且没有被任何其他注解标记的方法参数,
> 都被视为使用 @RequestParam 进行注解.

> 简单值类型的界定由BeanUtils.isSimpleProperty()确定.
> (ps: 总结下就是 基本类型及其包装类, 
> 还有 Enum, CharSequence, Nubmer, Date, URI, Locale. 以及它们的数组类型)

---

## @RequestHeader
## @CookieValue

## @ModelAttribut

这个注解会执行所谓的数据绑定,
```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { } 
```

// todo  数据绑定的流程...未完待续
上文的 Pet 实例将会解析如下:
1. 

> 注意 : 使用@ModelAttribut是可选的, 意思就是写不写随你. (ps: 和上面的@RequestParam类型呢) 
> 默认情况下任何不属于简单值类型的参数
> 并且没有被任何其他注解标记的方法参数,
> 都被视为使用 @ModelAttribute 进行注解.

> 简单值类型的界定由BeanUtils.isSimpleProperty()确定.
> (ps: 总结下就是 基本类型及其包装类, 
> 还有 Enum, CharSequence, Nubmer, Date, URI, Locale. 以及它们的数组类型)

## SessionAttributes

@SeesionAttribute 用于在Request之间 存储 model attributes 到 WebSession 中.

它是一个类基本的注解. 用于特定控制器使用的 session atributes.
通常列出模型属性的名称或模型属性的类型.

```java
@Controller
@SessionAttributes("pet")
public class PetController {
    //....
}
```

在第一次请求中, 当名称为pet的模型属性添加到模型中时,
它会自动提升并保存在WebSession中. 
它保持不变, 直到另一个控制器方法使用SessionStatus方法参数来清除存储.如下例所示：
```java
@Controller
@SessionAttributes("pet") 
public class PetController {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) { 
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete();
            // ...
        }
    }
}
```

## @RequestAttribute

## @RequsetBody
可以使用 @RequestBody 注解通过 HttpMessageReader 将 request 主体读取,
并反序列化为 Java 对象.

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```


# Model

# DataBinder
@Controller或@ControllerAdvice类可以有@InitBinder方法来初始化WebDataBinder的实例.

* 将请求参数(即表单数据或查询参数)绑定到模型对象.
* 将基于字符串的请求值转换为目标类型的控制器方法参数.
* 在呈现HTML表单时, 将模型对象格式化String值.
