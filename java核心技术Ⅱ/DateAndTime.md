# 日期和时间 API
Java 1.0 有一个 Date 类, 事后证明它过于简单了, 当 Java1.1 引入 Calendar 类之后,
Date 类中的大部分方法被弃用了. 但是, Calendar(日历) 的 API 还不够给力, 它的实例
是易变的, 并且它没有处理诸如闰秒这样的问题. 第三次升级是 Java1.8 中引入的 java.time
API, 它修正了过去的错误

# 时间线
Java 的 Date 和 Time API 规范要求 Java 使用的时间尺度为:
* 每天 86400 秒 (24小时)
* 每天正午与官方时间精确匹配
* 在其他时间点上, 以精确定义的方式与官方时间接近匹配

## Instant(瞬间)
Instant 表示时间线上的某个点
```java
//调用 static 方法 now() 可以获得当前时间点
Instant now = Instant.now();
```
## Duration(持续时间)
Duration 是两个时刻之间的时间量, 你可以通过调用 toNanos, toMillis, getSeconds, 
toMinutes, toHours, toDays 来获得 Duration 按照传统单位度量的时间长度
```java
//调用 Duration.between() 方法可以计算两个 Instant(时刻) 之间的时间量
Duration time = Duration.between(start, end);
```

# 本地时间
在 Java API 中有两种人类时间, 本地日期/时间 和 时区时间. 本地日期包含日期和当前的时间, 
但是与时区信息没有任何关联.

很多时间计算并不需要时区信息, 某些情况下, 时区甚至是一种障碍.

所以 API 的设计者们推荐程序员不要使用时区时间, 除非确实想要表示绝对时间的实例

LocalDate 是带有年, 月, 日的日期. 为了构建 LocalDate 对象, 可以使用 now 或 of 静态方法
# 格式化和解析
