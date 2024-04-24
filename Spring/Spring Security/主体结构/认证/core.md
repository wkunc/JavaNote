# Architecture Component (构建组件)

1. SecurityContextHolder
2. SecurityContext
3. Authentication
4. GrantedAuthority
5. ProviderManager
6. AuthenticationProvider
7. Request Credentials with AuthenticationEntryPoint
8. AbstractAuthenticationProcessingFilter

## SecurityContextHolder

Spring Security 认证模型的核心是 SecurityContextHolder.
它包含 SecurityContext

![](securitycontextholder.png)

Spring Security在SecurityContextHolder中存储用于通过认证的人员的详细信息.
Spring Security并不关心如何填充SecurityContextHolder.
如果它包含一个值, 那么它将用作当前通过身份验证的用户.

指示用户已经通过认证的最简单方法是直接设置SecurityContextHolder

```java
SecurityContext context = SecurityContexHolder.createEmptyContext();
Authentication authentication = new TestAuthenticationToken("username", "password", "ROLE_USER");
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context);
```

# GrantedAuthority (授予的权限)

`GrantedAuthority`是授予角色的高级权限.
role 或者 scopes 是最常见的例子
可以通过 Authentication.getAuthorities() 方法获得.

GrantedAuthoriity是授予 principal(主体)的权限, 此类权限通常是 role

# AuthenticationManager

负责处理一次 authentication request (认证请求).

```java
public interface AuthenticationManager {
    /*
    * 根据传入的 Authentication 对象, 进行验证,
    * 如果验证通过, 返回完整的 Authentication 对象(包含权限列表).
    * 遵从以下约定:
    * 1. 如果一个被禁用的账号进行验证, 那么必须抛出 DisabledException 异常
    * 2. 如果一个被锁定的账号进行验证, 那么必须抛出 LockedException
    * 3. 如果用错误的凭证(密码)进行验证, 那么必须抛出 BadCredentialsException
    */
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
}
```

# ProviderManager

Spring Security中的`AuthenticationManager`默认实现称为ProviderManager,
它不负责处理身份验证请求本身.
而是委托给内部的已配置的 `AuthenticationProvider` 列表.
每个验证提供者依次查询它们是否可以执行身份验证,
每个提供程序将抛出异常或返回完全填充的 Authentication 对象.

使用这种设计的原因是, 每个AuthenticationProvider都知道如何执行特定类型的身份验证.
例如: 一个AuthenticationProvider 可能可以验证 Username/Password, 而另一个可能可以验证SAML断言.
这允许每个AuthenticationProvider进行非常特定的身份验证, 同时支持多种身份验证, 并且只公开一个AuthenticationManager bean.

ProviderManager还允许配置可选的 parent AuthenticationManager,
如果当前没有可以执行的AuthenticationProvider, 就询问parent.

这种设计的最大目的就是保证灵活性

# AuthenticationProvider

可以将多个 AuthenticationProvider 注入 ProviderManager .
每个AuthenticationProvider执行特定的身份验证类型.
例如: DaoAuthenticationProvider 支持基于 username/password 形式的身份验证.
而 JwtAuthenticationProvider 支持对JWT令牌的身份验证.

处理身份验证请求的最常用方法是加载相应的UserDetails并检查加载密码与用户输入的密码是否一致.
这是 DaoAuthenticationProvider 使用的方法.
加载的 UserDetails 对象特别是它包含的权限列表会在构建完全填充的 Authentication 对象时使用.

Spring Security 提供了多个 AuthenticationProvider 实现来负责解析不同类型的 Authentication Token.
![](AuthenticationProvider.png)
我们主要学习 DaoAuthenticationProvider 这个具体实现类,
它负责认证的请求类型是 UsernamePasswordAuthenticationToken 及子类

# Authentication

![](authentication.png)
从 java.security.Principal 继续而来, 代表一次认证请求成功后的 token.

也就是说它是认证成功后返回的对象
(AuthenticationManager.authentication()方法的返回值),
但是它同时也被用于携带一次认证所需的信息
(AuthenticationManager.authentication()方法的入参).

```java
public interface Authentication extends Principal, Serializable {

    // 权限列表, 构建的认证请求时为空, 由AuthenticationManager设置, 以指示授予认证主体的权限
    // 当经过AuthenticationManager验证后应该时拥有一个完整的权限列表.
    // 重点: 实现应该确保对返回值的修改不会影响Authentication对象状态, 或使用不可修改的实例.
    // 如果返回的 Collection 是一个对象引用, 我们就可以向它添加或删除权限从而在运行时给主体赋予权限的能力.
    // 这是不安全的, 可能会被利用这个操作赋予普通用户过大的权力.
    // 所以要求修改返回值不会影响状态(说明返回值是一个副本集)或者返回值不可修改(这个没什么好说的Java集合提供的功能)
    //
    // 方便进行权限控制, 就是说如果一个方法, 路径需要某个权限才可以访问,
    // 我们只需要获取到经过认证的 Authentication 对象,
    // 通过它分析当前主体有无权限,就可以判断是否允许访问.
	Collection<? extends GrantedAuthority> getAuthorities();

    // 凭证, 或者说密码
	Object getCredentials();

    // 存储有关身份验证请求的其他详细信息, 可能时 IP 地址, 证书序列号等.
    // 除去 身份标识(账号), 凭证 (密码) 以外的额外细节信息放在这里.
	Object getDetails();

    // 主体, 通常是 UserDetails
	Object getPrincipal();

    // 是否进行认证. 我们创建的代表一个认证请求的实例对象应该是 false.
    // 代表它是没有被认证的. AuthenticationManager.authenticate()返回的对象实例
    // 应该是true 代表这个主体已经通过认证.
	boolean isAuthenticated();

	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

# Request Credentials with `AuthenticationEntryPoint`

AuthenticationEntryPoint用于发送HTTP响应, 以从客户端请求凭据.

# AbstractAuthenticationProcessingFilter

# Authentication Mechanisms (认证机制)

1. Username and Password
2. OAuth 2.0 Login
3. SAML 2.0 Login
4. Central Authentication Server (CAS)
5. Remember Me
6. JAAS Authentication
7. OpenID
8. Pre-Authentication Scenarios
9. X509 Authentication

### DaoAuthenticationProvider

学习这个类之前需要先学习其父类AbstractUserDetailsAuthenticationProvider.
这个抽象父类定义了如何与 UserDetails 交互, 但是没有定义如何加载UserDetails.
该类负责响应 UsernamePasswordAuthenticationToken 类型的身份验证请求.
验证成功后将创建 UsernamePasswordAuthenticationToken 并将其返回给调用者.
返回的令牌使用*用户名字符串*或*UserDetails*作为其主体(就是Token 中的 principal 字段会是字符串或UserDetails对象).
不过默认都是UserDetails(因为可以访问其他属性, 如电子邮件地址, 人性化的名称等).
可以通过设置boolean 标志**forcePrincipalAsString**来修改此行为.

通过存储放置在UserCache中的UserDetails对象来实现缓存功能,
这样可以确保可以验证具有相同用户名的后续请求, 而无需查询UserDetailsService.
应该注意的是, 如果用户似乎提供了错误的密码, 将查询UserDetailsService以确保比较的最新密码.
只有无状态应用程序才需要缓存. 例如: 在普通Web应用程序中, SecurityContext存储在Session中,
并不会在每个请求上重新验证用户. 因此, 默认缓存实现是NullUserCache(就是不缓存).

```java

public abstract class AbstractUserDetailsAuthenticationProvider implements
		AuthenticationProvider, InitializingBean, MessageSourceAware {

	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();

    // 用于实现缓存机制的接口实例.
	private UserCache userCache = new NullUserCache();
    // 两个控制行为的布尔标志,
    // 一个控制是否用String表示token中的主体,
    // 一个控制是否要暴露账号找不到异常(默认是不暴露, 防止被有心人利用, 会将账号找不到异常变成错误密码异常)
	private boolean forcePrincipalAsString = false;
	protected boolean hideUserNotFoundExceptions = true;

    // 两个用于先后检查Token的接口实例, 是同一个接口的实习, 只是使用的先后不同说明字段名不同.
    //
	private UserDetailsChecker preAuthenticationChecks = new DefaultPreAuthenticationChecks();
	private UserDetailsChecker postAuthenticationChecks = new DefaultPostAuthenticationChecks();
	private GrantedAuthoritiesMapper authoritiesMapper = new NullAuthoritiesMapper();

}
```

## 具体实现

所有具体类的抽象父类, 负责完成以下公有的功能.

1. 保存权限列表
2. 是否经过认证标志
3. 保存details对象
4. 实现Principal接口中的方法
5. 顺便实现了 CredentialsContainer 接口, 主要是考虑在认证完成后需要抹去具体的凭证信息(密码), 让密码信息持续存在系统中是不安全的.

除去上面这些明显的功能实现, 还通过构造器限定了子类的初始化逻辑.

```java
public abstract class AbstractAuthenticationToken implements Authentication,
		CredentialsContainer {

	private final Collection<GrantedAuthority> authorities;
	private Object details;
	private boolean authenticated = false;


    // 构造器, 必须传入一个权限集合. 实现了接口中关于权限列表的要求, 将列表设置为不可修改.
    // 由于列表不可修改, 又只有getter方法没有setter方法, 所以实际上所有的 Authentication 对象的权限必须在创建时设置.
    // 所以其实AuthenticationManager 并不是在传入的代表一个认证请求的 Authentication对象上添加权限而是重新new一个出来.
	public AbstractAuthenticationToken(Collection<? extends GrantedAuthority> authorities) {
		if (authorities == null) {
			this.authorities = AuthorityUtils.NO_AUTHORITIES;
			return;
		}

		for (GrantedAuthority a : authorities) {
			if (a == null) {
				throw new IllegalArgumentException(
						"Authorities collection cannot contain any null elements");
			}
		}
		ArrayList<GrantedAuthority> temp = new ArrayList<>(
				authorities.size());
		temp.addAll(authorities);
		this.authorities = Collections.unmodifiableList(temp);
	}

    // 实现抹去 Credentials 功能.
	public void eraseCredentials() {
		eraseSecret(getCredentials());
		eraseSecret(getPrincipal());
		eraseSecret(details);
	}

    // 子类负责实现的抽象方法.
    Object getCredentials();
    Object getPrincipal();

}
```

具体子类比较多, 但大多数都比较简单.
这里以UernamePasswrodAuthenticationToken为例.

实现抽象方法非常简单, 用两个Object 字段保存一下就可以了.
主要是提供了两个构造器 : 一个用于构建认证请求, 一个用于构建通过认证后的 token.
还有就是覆盖了父类的 setAuthenticated() 方法,
这样没有人可以对一个没有认证的实例以调用方法 setAuthenticated(true) 的方式来将其标记为已认证,
从而绕过AuthenticationManager 认证机制.

```java
public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {

	private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;


	private final Object principal;
	private Object credentials;


    // 用于构建一个认证请求的构造器, 主要是没有设置权限列表和 authenticated 字段会被设置为 false;
	public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		super(null);
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false);
	}

    // 构建一个完整的Authentication对象, 这个构造器只应该被 AuthenticationManager 调用.
    // 这里只是说明正确的逻辑, 并没有什么措施来阻止我们调用.
    // 后面我们可以看到其实 权限管理方面 并不再乎Authentication对象是怎么来的, 只在乎主体是否又对应权限.
	public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
			Collection<? extends GrantedAuthority> authorities) {
		super(authorities);
		this.principal = principal;
		this.credentials = credentials;
		super.setAuthenticated(true); // must use super, as we override
	}

	// ~ Methods
	// ========================================================================================================

	public Object getCredentials() {
		return this.credentials;
	}

	public Object getPrincipal() {
		return this.principal;
	}

	public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
		if (isAuthenticated) {
			throw new IllegalArgumentException(
					"Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
		}

		super.setAuthenticated(false);
	}

	@Override
	public void eraseCredentials() {
		super.eraseCredentials();
		credentials = null;
	}
}
```

## AuthenticationManager的实现.

Spring Security中的默认实现称为ProviderManager, 它不负责处理身份验证请求本身,
而是委托给已配置的 AuthenticationProvider 列表.
每个验证提供者依次查询它们是否可以执行身份验证,
每个提供程序将抛出异常或返回完全填充的 Authentication 对象.

处理身份验证请求的最常用方法是加载相应的UserDetails并检查加载密码与用户输入的密码是否一致.
这是 DaoAuthenticationProvider 使用的方法.
加载的 UserDetails 对象特别是它包含的权限列表会在构建完全填充的 Authentication 对象时使用.

Spring Security 提供了多个 AuthenticationProvider 实现来负责解析不同类型的 Authentication Token.
![](AuthenticationProvider.png)
我们主要学习 DaoAuthenticationProvider 这个具体实现类,
它负责认证的请求类型是 UsernamePasswordAuthenticationToken 及子类

### DaoAuthenticationProvider

学习这个类之前需要先学习其父类AbstractUserDetailsAuthenticationProvider.
这个抽象父类定义了如何与 UserDetails 交互, 但是没有定义如何加载UserDetails.
该类负责响应 UsernamePasswordAuthenticationToken 类型的身份验证请求.
验证成功后将创建 UsernamePasswordAuthenticationToken 并将其返回给调用者.
返回的令牌使用*用户名字符串*或*UserDetails*作为其主体(就是Token 中的 principal 字段会是字符串或UserDetails对象).
不过默认都是UserDetails(因为可以访问其他属性, 如电子邮件地址, 人性化的名称等).
可以通过设置boolean 标志**forcePrincipalAsString**来修改此行为.

通过存储放置在UserCache中的UserDetails对象来实现缓存功能,
这样可以确保可以验证具有相同用户名的后续请求, 而无需查询UserDetailsService.
应该注意的是, 如果用户似乎提供了错误的密码, 将查询UserDetailsService以确保比较的最新密码.
只有无状态应用程序才需要缓存. 例如: 在普通Web应用程序中, SecurityContext存储在Session中,
并不会在每个请求上重新验证用户. 因此, 默认缓存实现是NullUserCache(就是不缓存).

```java

public abstract class AbstractUserDetailsAuthenticationProvider implements
		AuthenticationProvider, InitializingBean, MessageSourceAware {

	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();

    // 用于实现缓存机制的接口实例.
	private UserCache userCache = new NullUserCache();
    // 两个控制行为的布尔标志,
    // 一个控制是否用String表示token中的主体,
    // 一个控制是否要暴露账号找不到异常(默认是不暴露, 防止被有心人利用, 会将账号找不到异常变成错误密码异常)
	private boolean forcePrincipalAsString = false;
	protected boolean hideUserNotFoundExceptions = true;

    // 两个用于先后检查Token的接口实例, 是同一个接口的实习, 只是使用的先后不同说明字段名不同.
    //
	private UserDetailsChecker preAuthenticationChecks = new DefaultPreAuthenticationChecks();
	private UserDetailsChecker postAuthenticationChecks = new DefaultPostAuthenticationChecks();
	private GrantedAuthoritiesMapper authoritiesMapper = new NullAuthoritiesMapper();

}
```

```java
public Authentication authenticate(Authentication authentication)
        throws AuthenticationException {
    // 确保传入的Token对象是 UsernamePasswordAuthenticationToken 类型的.
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
            () -> messages.getMessage(
                    "AbstractUserDetailsAuthenticationProvider.onlySupports",
                    "Only UsernamePasswordAuthenticationToken is supported"));

    // Determine username
    // 1. 明确用户名, 传入的Authentication对象代表请求, 所以其中的 Object Principal 字段通常是String username
    String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
            : authentication.getName();

    // 2. 从缓存中获取, 这个布尔值代表当前 user 变量是从哪里获取来的, true 代表从cache中来, false 代表是查询信息存储库.
    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);

    // 3. 如果没有获取到, 就调用 retrieveUser()抽象方法, 从存储库中获取, 子类实现就是从内部的 UserDetailsService 中获取.
    // 顺便处理 UsernameNotFoundException , 根据设置选择是否转换为 BadCredentialsException 异常, 还有将之前的 cacheWasUsed 表示设置为false
    if (user == null) {
        cacheWasUsed = false;

        try {
            // 从任何地方加载 UserDetails 对象, 由子类实现.
            // 返回值不能null, 这是接口约定, 找不到用户抛出就抛出UsernameNotFoundException.
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (UsernameNotFoundException notFound) {
            logger.debug("User '" + username + "' not found");

            if (hideUserNotFoundExceptions) {
                throw new BadCredentialsException(messages.getMessage(
                        "AbstractUserDetailsAuthenticationProvider.badCredentials",
                        "Bad credentials"));
            }
            else {
                throw notFound;
            }
        }

        Assert.notNull(user,
                "retrieveUser returned null - a violation of the interface contract");
    }

    // 4. 无论UserDetails 是从哪里获取的都要经过下面两个检查.
    // 第一个检查是预先检查, 它会检查UserDetails中的 AccountNonLocked, Enable, AccountNoExpired 三个布尔值,
    // 确定是否要抛出对应异常, LockedException, DisabledException, AccountExpiredException.
    // 意味着如果UserDetails中的着三个信息是false的话, 就连密码匹配都没有走到.
    // 第二个 additionalAuthenticationChecks() 方法是抽象方法, 交给子类实现,
    // 子类在这个方法中进行了密码匹配工作匹配失败会抛出BadCredentialsException异常.
    try {
        preAuthenticationChecks.check(user);
        additionalAuthenticationChecks(user,
                (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (AuthenticationException exception) {
        // 当捕捉到AuthenticationException异常时, 并且cacheWasUsed表示为true.
        // 代表第一步从缓存中取得的UserDetails是错误的, 所以要重新从存储库中获取, 然后经过下面两个方法的检查.
        if (cacheWasUsed) {
            // There was a problem, so try again after checking
            // we're using latest data (i.e. not from the cache)
            cacheWasUsed = false;
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
            preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        else {
            throw exception;
        }
    }

    // 然后都会执行最后的检查, 这个会检查 CredentialsNonExpired 信息.
    // 如果凭证被标记为过期的, 即使上面的凭证匹配通过也没有用, 还是认证失败了.
    postAuthenticationChecks.check(user);

    // 判断最后的结果是否是从存储库中获取的.
    // 如果是那么会在缓存中保存一份方便重复认证.
    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }

    // 准备返回的完整的Token
    // 1. 确定是以 UserDetails 作为返回Token的主体还是String
    Object principalToReturn = user;

    if (forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }

    // 根据返回Token的主体, 查询出的UserDetails, 以及传入的 认证请求构建完整的Token
    return createSuccessAuthentication(principalToReturn, authentication, user);
}

// 注意这个方法会被子类重写, 加入将密码在进行加密的操作.
protected Authentication createSuccessAuthentication(Object principal,
        Authentication authentication, UserDetails user) {
    // Ensure we return the original credentials the user supplied,
    // so subsequent attempts are successful even with encoded passwords.
    // Also ensure we return the original getDetails(), so that future
    // authentication events after cache expiry contain the details
    UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
            principal, authentication.getCredentials(),
            authoritiesMapper.mapAuthorities(user.getAuthorities()));
    result.setDetails(authentication.getDetails());

    return result;
}
```

## 认证机制


