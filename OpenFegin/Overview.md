# OpenFeign
Feign is a Java to HTTP client binder inspired by Retrofit, JAXRS-2.0, and WebSocket.
Feign's first goal was reducing the complexity of binding Denominator uniformly to HTTP APIs regardless of ReSTfulness.

Feign 是受到 Retrofit, JAXRS-2.0 和 WebSokcet 启发的.
Java 到 HttpClient 端绑定程序. Feign的目标是减少与Restfulness 无关的的事情.

# Interface Annotations

| Annotation   | Interface Target | usage                                                                                                      |
|--------------|------------------|------------------------------------------------------------------------------------------------------------|
| @RequestLine | Method           | http请求URL模板, 如@RequestLine("Get /repos/{ower}/repo/contributors"), 可以用{expression}的形式引用方法参数中的 @Param 标注的参数 |
| @Param       | Parameter        | 标注参数的名字, 用于@RequestLine, @Headers, @Body 等模板中.                                                             |
| @Headers     | Method, Type     | http请求 header 模板, 可以添加各种请求头, 也可以引用 @Param. 可以写在接口上, 作为预请求头.                                                |
| @QueryMap    | Parameter        | 写在方法参数上, 表示对应的参数应该变成 http 查询参数.                                                                            |
| @HeaderMap   | Parameter        | 对应的参数会被变成 http header 的一部分.                                                                                |
| @Body        | Method           | 请求body模板.                                                                                                  |

# 模板与表达式
Fegin 的 `expression` 是 简单字符串表达式 (Level 1), 由 URI Template - RFC 6570 定义
被@Param注解的方法参数, 扩展了表达式.

```java
public interface Github {
    // RequestLine(请求URL模板), @Headers(请求头模板), @Body(请求体模板).
    //  等模板总可以引用, 被@Param标记的方法参数, 来完成一个比较动态的请求构造.
    @RequestLine("GET /repos/{owner}/{repo}/contributors")
    List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repository);

    class Contributor {
        String login;
        int contributions;
    }

}
```

## Request Parameter Expansion

@RequestLine, @QuerMap 模板遵循URL模板-RFC6570标准, 用于一级模板

1. 未解析的表达式将被省略
2. 如果尚未通过 @Param 注解标记为已经编码, 默认所有的文字和变量值都会经过 pct 编码.

> Undefined vs. Empty Values
> 未定义的表达式是表达式的值是null, 或者没有提供值.
> 根据RFC-6570, 可以为表达式提供一个空值.
> 当Feign解析表达式时, 它首先确定是否定义了值, 如果是, 则保留查询参数.
> 如果表达式未定义, 则删除查询参数.

Empty String
```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", "");
   this.demoClient.test(parameters);
}
// Result
// http://localhost:8080/test?param=
```

Missing
```
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   this.demoClient.test(parameters);
}

// Result
// http://localhost:8080/test
```

Undefined
```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", null);
   this.demoClient.test(parameters);
}
// http://localhost:8080/test
```

## Request Header Expansion


## Body Expansion


# Customization (自定义)
Feign 有一个可以定制的方面, 对于简单的情况, 
可以使用 Feign.builder() 来使用自定义组件构造API接口.

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}

public class BankService {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
    .decoder(new AccountDecoder())
    .target(Bank.class, "https://api.examplebank.com");
  }
}
```


