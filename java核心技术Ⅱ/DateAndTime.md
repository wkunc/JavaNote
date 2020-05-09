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
Instant 表示时间线上的某个点. 可以理解为时间戳, 或者说它就是时间戳

时间线的原点被设置为格林威治天文台的子午线所在时区的 1970.1.1 的午夜.
这与 UNIX/POSIX 时间中的惯例相同. 从原地开始, 时间按照每天 86400 秒向前或向后度量, 精确到纳秒.

Instant 的值可以追溯 `10亿年(Instant.MIN)`.这对于表示宇宙年龄(135亿年)还差的远. 
但对于实际应用来说应该够了. 最大值时 Instant.MAX 是公元 1000 000 000 年的12月31日.



下面是常用的静态方法, 用于直接获取指定的时间点.
```java
public static Instant from(TemporalAccessor temporal)

//调用 static 方法 now() 可以获得当前时间点
public static Instant now()

public static Instant now(Clock clock)

// 从时间原点开始计算时间点. 单位毫秒
public static Instant ofEpochMilli(long epochMilli)

// 从时间原点开始计算时间点. 单位秒
public static Instant ofEpochSecond(long epochMilli)

public static Instant ofEpochSecond(long epochMilli, long nanoAdjustment)

// 从类似 2007-12-03T10:15:30.00Z 中的时间文本中解析出时间点.
static Instant parse(CharSequence text)
```

实例方法主要提供基于时间戳的加减运算. 以及时间点之间的比较
以及与`Time API`中的其他表示类型进行转换.

## Duration(持续时间)
Duration 是两个时刻之间的时间量, 你可以通过调用 toNanos, toMillis, getSeconds, 
toMinutes, toHours, toDays 来获得 Duration 按照传统单位度量的时间长度
```java
//调用 Duration.between() 方法可以计算两个 Instant(时刻) 之间的时间量
Duration time = Duration.between(start, end);
```

# 本地时间 (LocalDate/LocalTime/LocalDateTime)
在 Java API 中有两种人类时间, 本地日期/时间 和 时区时间.
LocalDateTime 包含日期和当前的时间, 但是与时区信息没有任何关联.

很多时间计算并不需要时区信息, 某些情况下, 时区甚至是一种障碍.

所以 API 的设计者们推荐程序员不要使用时区时间, 除非确实想要表示绝对时间的实例

LocalDate 是带有年, 月, 日的日期. 为了构建 LocalDate 对象, 可以使用 now 或 of 静态方法

```java
public static LocalDate now()

public static LocalDate now(Clock clock)

public static LocalDate of(int yer, int month, int dayOfMonth)

public static LocalDate of(int yer, Month month, int dayOfMonth)

//
public static ofInstant(Instant instant, ZoneId zone)

// 从给定年份的日期中获取
public static LocalDate ofYearDay(int year, int dayOfYear)

// 解析String类型的日期文本
public static LocalDate parse(CharSequence text)
public static LocalDate parse(CharSequence text, DateTimeFormatter formatter)
```

# 格式化和解析


# 和遗留代码进行转换.

Date 类似 Instant. 所以Java8中为Date添加了两个新方法
方便转换为 Instant 以及从 Instant 变为 Date
