# Locale 对象
有若干个专门负责格式处理的类. 为了对格式化进行控制, 可以使用 Locale 类.
locale 由多达 5 个部分组成:
1. 一种语言
2. 可选的一段脚本
3. 可选的一个国家或地区
4. 可选的一个变体, 用于指定各种杂项特性
5. 可选的一个扩展

locale 是用标签描述的, 标签是由 locale 各个元素通过连字符连接起来的字符串, 如:
en-US, zh-CN

我们可用标签字符来创建 Locale 对象
```java
Locale usEnglish = Locale.forLanguageTag("en-US");
```
toLanguageTag 方法可以生成给定 Locale 的语言标签

为了方便 JavaSE 为各个国家预定义了 Locale 对象:
* Locale.CANADA
* Locale.CHINA
* locale.UK
* locale.US
等等

# 数字格式
之前我们通过国家语言等要素确定了一个 Locale 对象
然后, 由于数字和货币是的格式是高度依赖 Locale 的.
所以 Java 类库提供了一个格式化和解析器.
你可以通过下面步骤对特定 Locale 的数字进行格式化
1). 获得一个 Locale 对象
2). 使用一个 factory-method 获得一个 格式器对象
3). 使用这个 格式器对象 来完成格式化和解析工作

factory方法是 NumberFormat 类的静态方法, 它们接受一个 Locale 类型的参数.
一共由三个工厂方法: getNumberInstance(), getCurrencyInstance(), getPercentInstance().
这些方法返回的 NumberFormat(其实是它的子类) 分别可以对数字, 货币, 百分比进行格式化和解析.

