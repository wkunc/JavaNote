# Authorization Architecture

## Authorities

Authentication(认证), 讨论所有的身份验证实现如何存储 GrantedAuthority 对象列表.
这些代表以授予委托人的权限.
GrantedAuthority 对象由 AuthenticationManager 插入到 Authentication对象
并在以后作出授权决策时由 AccessDecisionManager 读取

## Pre-Invocation Handling

Spring Security 提供了拦截器, 用于控制安全对象的访问, 例如方法调用或web请求.
AccessDecisionManager 会做出关于是否允许进行调用的调用前决定.

### AccessDecisionManager

```java
public interface AccessDecisionManager {


    /**
    * Resolves an access control decision for the passed parameters
    */
    void decide(Authentication authentication, Object object,
            Collection<ConfigAttribute> configAttributes) throws AccessDeniedException,
            InsufficientAutheticationException;

    boolean supports(ConfigAttribute attribute);


    boolean supports(Class<?> clazz);

}
```

### Voting-Based AccessDecisionManager Implementations

基于选票的 AccessDecisionManager 实现



