# Authentication Sequence (认证流程)
## Step 1
调用 Subject.login 方法传递 AuthenticationToken 实例, 它代表 User 的 principals(主体,账号,id) 和 credentials(凭证,密码)
```java
Subject currentUser = SecurityUtils.getSubject();
if (!currentUser.isAuthenticated()) {
        UsernamePasswordToken token = new UsernamePasswordToken("lonestarr", "vespa");
        token.setRememberMe(true);
        try {
            //调用 Subject.login()方法
            currentUser.login(token);
        } catch (UnknownAccountException uae) {
            log.info("There is no user with username of " + token.getPrincipal());
        } catch (IncorrectCredentialsException ice) {
            log.info("Password for account " + token.getPrincipal() + " was incorrect!");
        } catch (LockedAccountException lae) {
            log.info("The account for username " + token.getPrincipal() + " is locked.  " +
                    "Please contact your administrator to unlock it.");
        }
        // ... catch more exceptions here (maybe custom ones specific to your application?
        catch (AuthenticationException ae) {
            //unexpected condition?  error?
        }
    }
```
## Step 2
Subject 实例(典型的是一个 DelegatingSubject,或它的子类) 通过调用 securityManager.login(token) 委托给 Application 中的
SecurityManager, 真正的身份验证开始
```java
//DelegatingSubject 类中 login()方法
public void login(AuthenticationToken token) throws AuthenticationException {
    clearRunAsIdentitiesInternal();
    //重点,真正的 认证流程从这里开始
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        //we have to do this in case there are assumed identities - we don't want to lose the 'real' principals:
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    this.principals = principals;
    this.authenticated = true;
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```

## Step 3
SecurityManager 接收 token 然后简单的 delegates (代理) 它内部的 Authenticator (认证器) 实例, 调用认证器的认证方法
Authericator 一般是 ModularRealmAuthenticator (模块化 Realm 认证器)实例, 它支持多Realm验证 
```java
    //DefaultSecurityManager 类中的 login()
    public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {
            //调用了自己从 AuthenticatingSecurityManager 继承来的方法
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
                if (log.isInfoEnabled()) {
                    log.info("onFailedLogin method threw an " +
                            "exception.  Logging and propagating original AuthenticationException.", e);
                }
            }
            throw ae; //propagate
        }

        Subject loggedIn = createSubject(token, info, subject);

        onSuccessfulLogin(token, info, loggedIn);

        return loggedIn;
    }
    // 这就是那个继承来的方法,就是简单的调用内部的 Authenticator 的 authenticate()方法
    public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
        return this.authenticator.authenticate(token);
    }
```

## Step 4
Authericator 如果有多个 Realm 就根据 AuthenticationStrategy(认证策略)执行认证, 如果是单个 Realm 就没必要使用认证策略
```java
    //
    protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
        assertRealmsConfigured();
        Collection<Realm> realms = getRealms();
        if (realms.size() == 1) {
            return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
        } else {
            //多 Realm 验证使用策略
            return doMultiRealmAuthentication(realms, authenticationToken);
        }
    }
```

## Step 5
查询每个 Realm 查看它是否支持提交的 Token. 如果支持调用 Realm 的 getAuthenticationInfo()方法

## Save Time
实现 Realm 接口是困难的,大多数人都选择继承 abstract AuthorizingRealm 类, 这个类实现了共同的 认证 和 授权流程

# Credntials Matching(凭证匹配)

