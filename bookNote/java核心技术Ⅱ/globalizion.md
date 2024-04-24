# Java 对国际化(i18n)的支持

感觉重点是资源包(Resource bundle)的使用.

# Locale 对象

有若干个专门负责格式处理的类. 为了对格式化进行控制, 可以使用 Locale 类.
locale 由多达 5 个部分组成:

1. 一种语言 : Chinese zh, Japanes ja, English en, Korean ko....
2. 可选的一段脚本 : 由首字母大写的4个字母表示如: Latn(拉丁文), Cyrl(西里尔文), Hant(繁体中文). 有些语言可以使用多种方式书写比如
   中文有简体和繁体.
3. 可选的一个国家或地区 :
4. 可选的一个变体, 用于指定各种杂项特性, 例如方言和拼写规则. (变体已经很少使用, 很多变体被归类成了后面的扩展)
5. 可选的一个扩展. 扩展描述了日历(比如说日本历)和数字(代替西方数字的泰语数字)等内容的本地偏好.

locale 是用标签描述的, 标签是由 locale 各个元素(上面提到的5个)通过连字符连接起来的字符串, 如:
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
  ....
  还有预定义了大量语言的Local
* Locale.CHINESE
* Locale.ENGLISH
  ....
  这些内容都在Local中以常量的方式存在.

# 数字格式

之前我们通过国家语言等要素确定了一个 Locale 对象
然后, 由于数字和货币是的格式是高度依赖 Locale 的.
所以 Java 类库提供了一个格式化和解析器.
你可以通过下面步骤对特定 Locale 的数字进行格式化

1. 获得一个 Locale 对象
2. 使用一个 factory-method 获得一个 格式器对象
3. 使用这个 格式器对象 来完成格式化和解析工作

factory方法是 NumberFormat 类的静态方法, 它们接受一个 Locale 类型的参数.
一共由三个工厂方法: getNumberInstance(), getCurrencyInstance(), getPercentInstance().
这些方法返回的 NumberFormat(其实是它的子类) 分别可以对数字, 货币, 百分比进行格式化和解析.

# 货币格式

# 日期格式

# 消息格式化

Java 类库中有一个 MessageFormat 类, 它与用 printf 方法进行格式化很类似, 但是它支持 Locale,
并且会对数字和日期进行格式化.

## 格式化数字和日期

# 资源包

## 定位资源包

当本地化一个应用时, 会产生很多资源包(resource bundle). 就像游戏会出许多的语言版本.
每一个包都是一个属性文件或者是一个描述了locale相关的类(消息, 标签)

需要对这些包使用一种同一的命名规则, 例如:
为中国定义的资源放在一个名为 "包名_zh_CN",
而为所有说中文的所共享的资源放在 "包名\_zh"的文件中.

一般来说使用:
包名_语言_国家 格式的文件来命名所有和国家有关的资源.
包名\_语言 格式
