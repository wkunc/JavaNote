# Tagging and Filtering
测试类和方法可以被``@Tage``注解标记, 这些标记在稍后可以被``filter``过滤执行.

## Syntax Rules for Tags(标签的语法规则)
* 一个标签不能是``null``或者是``blank``
* 修剪的标签不得包含空格
* 修剪的标签不得包含ISO控制字符
* 修剪的标签不得包含以下任何保留字符
 * ,
 * (和)
 * &
 * |
 * !


```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Repeatable(Tags.class)
@API(status = STABLE, since = "5.0")
public @interface Tag {

	/**
	 * The <em>tag</em>.
	 *
	 * <p>Note: the tag will first be {@linkplain String#trim() trimmed}. If the
	 * supplied tag is syntactically invalid after trimming, the error will be
	 * logged as a warning, and the invalid tag will be effectively ignored. See
	 * {@linkplain Tag Syntax Rules for Tags}.
	 */
	String value();

}
```

