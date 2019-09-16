# AccessDecisionManager
在Spring Security中负责进行访问控制决策的主要接口是AccessDecisionManager.
它有一个 decide(决定)方法, 它接受一个表示请求访问主体的Authentication对象,
一个 securityObject, 和一个应用该对象的安全元数据列表(比如说: 访问所需的角色列表).

```java

public interface AccessDecisionManager {
	/**
	 * 做一个访问控制决定
	 *
	 * @param authentication 访问者的表示.
	 * @param 将被访问的安全对象
	 * @param configAttributes 一个和被调用安全对象相关的配置(比如我们配置的需要的角色列表)
	 *
	 * @throws AccessDeniedException 当访问者不具备权限访问时抛出异常 
	 * @throws InsufficientAuthenticationException 当代表访问者的Authtication级别不够时抛出异常
     * (比如说修改密码需要完全登陆状态也就是UsernamePasswordAuthenticationToken,
     * 而当前用户是通过记住我功能认证的也就是RememberMeAuthenticationToken, 
     * 此时认证的级别不够也一样不给访问, 只不过原因不同) 
	 */
	void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException,
			InsufficientAuthenticationException;

	/**
	 * Indicates whether this <code>AccessDecisionManager</code> is able to process
	 * authorization requests presented with the passed <code>ConfigAttribute</code>.
	 * <p>
	 * This allows the <code>AbstractSecurityInterceptor</code> to check every
	 * configuration attribute can be consumed by the configured
	 * <code>AccessDecisionManager</code> and/or <code>RunAsManager</code> and/or
	 * <code>AfterInvocationManager</code>.
	 * </p>
	 *
	 * @param attribute a configuration attribute that has been configured against the
	 * <code>AbstractSecurityInterceptor</code>
	 *
	 * @return true if this <code>AccessDecisionManager</code> can support the passed
	 * configuration attribute
	 */
	boolean supports(ConfigAttribute attribute);

	/**
	 * Indicates whether the <code>AccessDecisionManager</code> implementation is able to
	 * provide access control decisions for the indicated secured object type.
	 *
	 * @param clazz the class that is being queried
	 *
	 * @return <code>true</code> if the implementation can process the indicated class
	 */
	boolean supports(Class<?> clazz);
```

# 基于Voter的实现
我们可以实现自己的AccessDecisionManager来控制的所有细节,
但是 SpringSecurity 提供了几个基于(Voting)投票的 AccessDecisionManager 实现.
![](AccessDecisionManager.png)

这个实现, 将对授予决策轮询一系列的AccessDecisionVoter实现.
然后, 根据每个Voter(选民)的投票进行评估决定是否拒绝访问(抛出AccessDeniedException)

```java
// 接口方法级别和AccessDecisionManager一样. 
// 只是decis()方法变成的 vote(), 因为基于投票的访问控制器需要综合所有的选民(Voter)的结果.
// 所以这里是具有返回值的. 返回值的约定是 -1 否定, 0 弃权, 1 通过. 分别在这个接口中以静态字段的形式存在.
// 如果需要自己实现这个选民接口, 记得要返回合适的值.
public interface AccessDecisionVoter<S> {
    int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attrs);
    boolean supports(ConfigAttribute attribute);
    boolean supports(Class clazz);
}
```
> 这个设计和认证管理器默认实现(AuthenticationManager)有类似的地方,
> 一样是将过程委托给内部的子接口(AuthenticationProvider), 方便扩展.

Spring Security 提供了三个不同策略的实现.
为什么又这么多呢, 因为不同的策略对同一投票结果又不同的决策.
1. 只要有Voter投了 GRANTED(通过) 就可以访问, 不管是否有Voter投否定票.
2. 只有所有的Voter投了通过才可以访问, 换种说明就是, 只要有Voter投了否定就拒绝方法(和上面刚好相反)
3. 看选票结果, 如果投通过的Voter多就允许访问, 反之如果投否定的Voter多就拒绝访问

当然还有弃权的情况.上面这三种策略对弃权的处理分别是:
1. 投票结束还没有执行完只有两种可能(一. 有人投了反对票, 二. 所有人都弃权),
如果没有人投肯定票, 而有人投了反对票 那就拒绝访问.
如果大家都弃权那么根据一个是否允许全部弃权访问的boolean标志位控制默认是false.
2. 如果没有人投反对, 而有人投了肯定, 那么就是允许访问, 如果大家都弃权那么和上面一样由系统设置决定.
3. 
总之基于Voter的实现还是很简单的(忘记的话可以自己看代码).


