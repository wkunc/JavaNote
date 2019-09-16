# UrlPathHelper
第一次是在 AbstractHandlerMapping 中见到,
这个帮助类的目的是确定请求的路径. 以及对url进行编码操作.
```java
public class UrlPathHelper {

	private static final String WEBSPHERE_URI_ATTRIBUTE = "com.ibm.websphere.servlet.uri_non_decoded";

	private static final Log logger = LogFactory.getLog(UrlPathHelper.class);

	@Nullable
	static volatile Boolean websphereComplianceFlag;


    // 是否启用完全匹配, 即包含Context路径
	private boolean alwaysUseFullPath = false;

    // 是否编码
	private boolean urlDecode = true;

    // 是否移除分号内容, 如: /sping/adfas;JSessionID=sdfsla....
	private boolean removeSemicolonContent = true;

    // 编码采用的默认编码集 ISO-8859-1
	private String defaultEncoding = WebUtils.DEFAULT_CHARACTER_ENCODING;
}
```

最常见的方法调用是
```java
// 根据一个简单的逻辑判断应该返回怎么样的路径.
// 首先如果是采用完全路径的话, 就调用 getPathWithinApplication() 方法.
// 会返回如 :/spring_war/spring/test/path/1 (/context_path/servlet_path/...)
// 就是返回完整的请求路径. 当然会进行URL的编码, 去除重复斜线, 以及分号内容等工作.
// 如果是不要求返回完全路径的话, 就调用 getPathWithinServletMapping() 方法.
// 主要是需要将完全路径中的 context_path, 和 servlet_path 除去, 留下剩余的
// 如: /test/path/1 , 一样会执行 URL 编码, 重复斜线去除, 分号内容...
public String getLookupPathForRequest(HttpServletRequest request) {
    if (this.alwaysUseFullPath) {
        return getPathWithinApplication(request);
    }
    // Else, use path within current servlet mapping if applicable
    String rest = getPathWithinServletMapping(request);
    if (!"".equals(rest)) {
        return rest;
    }
    else {
        return getPathWithinApplication(request);
    }
}
```

# PathMatcher
是一个策略接口, Spring 只提供了一个 AntPahtMatcher 实现 Ant 风格的路径匹配.
```java
public interface PathMatcher {

    // 返回给定的path是否是一个模式, 判断字符串是否是Ant风格
	boolean isPattern(String path);

	boolean match(String pattern, String path);

	boolean matchStart(String pattern, String path);

	String extractPathWithinPattern(String pattern, String path);

	Map<String, String> extractUriTemplateVariables(String pattern, String path);

	Comparator<String> getPatternComparator(String path);

	String combine(String pattern1, String pattern2);
}
```

最简单能看到的地方是 AbstractUrlHandlerMapping 系列中
(根据Url直接匹配的HandlerMapping).



