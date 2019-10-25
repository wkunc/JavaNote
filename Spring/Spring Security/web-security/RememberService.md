# RememberMeService

```java
public interface RememberMeServices {

    // 从请求中获取remember cookie 生成 Authentication 对象, 用于Remember-Me认证.
	Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);

	void loginFail(HttpServletRequest request, HttpServletResponse response);

	void loginSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication successfulAuthentication);
}
```

