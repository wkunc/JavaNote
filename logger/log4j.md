# 整体逻辑

首先确定使用的实现(通过属性文件指定和SPI注册)
getLogger()方法会调用 getContext() 方法.
内部会使用确定的 LoggerContextFactory 实例创建Context.
使用获得的Context来创建Logger.
LoggerContextFactory 中创建Context的又是一个 ContextSelector.

# LogManager

```java
public class LogManager {
    // 没有成员变量, 只有一些静态的常量
    public static final String FACTORY_PROPERTY_NAME = "log4j2.loggerContextFactory";
    public static final String ROOT_LOGGER_NAME = String.EMPTY;

    private static final Logger LOGGER = StatusLogger.getLogger();
    private static fianl String FQCN = LogManager.class.getName();
    private static volatile LoggerContextFactory factory;

    // 子类访问权限的构造器, 保护这个类
    protected LogManager() {
    }
}
```

1. 根据 log4j2.component.properties 文件中的 log4j2.loggerContextFactory 属性值确定指定的 LoggerContextFactory 实现类.
2. 用SPI机制加载 Provider 实现类和 MTA-INF/log4j-provider.properties 确定.
3. 如果上面都失败的话就使用 SimpleLoggerContextFactory.

```java
// 加载逻辑都在static加载块中
// 主要目的是: 初始化 factory 字段, Log4jContextFactory.
// 具体步骤为 :
// 1. 检测 log4j2.component.properties 文件中的 log4j2.loggerContextFactory 指定的实现类.
// 如果有指定的实现, 那么尝试加载Class, 然后通过默认构造器实例化.(所以说这里要 LoggerContextFactory 接口的实现类必须有无参构造器)
// 如果上面没有指定, 或者说在尝试初始化指定的实现类失败了, 那么继续执行.
// 2. 使用ProviderUtil来进行加载Provider接口的实例, ProviderUtil是一个懒加载的单例.
//    会在第一次初始化时会使用JDK的SPI机制加载Provider.class的实现.
//    (然后log4j-core包中就有一个对应SPI文件, 里面指定了实现类 Log4jContextFactory)
//
// 3. 如果这样都加载失败了(说明log4j-core不在classpath中, 也没有第三方实现).
//    那么会使用一个 SimpleLoggerContextFactory(log4j-api自带的简单实现) 作为最后的选择.
static {
    // 根据 log4j2.component.properties 文件中的 log4j2.loggerContextFactory 属性值确定实现类.
    final PropertiesUtil managerProps = PropertiesUtil.getProperties();
    final String factoryClassName = managerProps.getStringProperty(FACTORY_PROPERTY_NAME);
    if (factoryClassName != null) {
        try {
            factory = LoaderUtil.newCheckedInstanceOf(factoryClassName, LoggerContextFactory.class);
        } catch (final ClassNotFoundException cnfe) {
            LOGGER.error("Unable to locate configured LoggerContextFactory {}", factoryClassName);
        } catch (final Exception ex) {
            LOGGER.error("Unable to create configured LoggerContextFactory {}", factoryClassName, ex);
        }
    }

    // 如果 factory == null 说明上面使用 log4j2.component.properties 属性文件指定的方式失败了.
    if (factory == null) {
        final SortedMap<Integer, LoggerContextFactory> factories = new TreeMap<>();
        // 这里使用了 ProviderUtil 来使用 SPI 机制和 MTEA-INF/log4j-provider.properties.
        if (ProviderUtil.hasProviders()) {
            for (final Provider provider : ProviderUtil.getProviders()) {
                final Class<? extends LoggerContextFactory> factoryClass = provider.loadLoggerContextFactory();
                if (factoryClass != null) {
                    try {
                        factories.put(provider.getPriority(), factoryClass.newInstance());
                    } catch (final Exception e) {
                        LOGGER.error("Unable to create class {} specified in provider URL {}", factoryClass.getName(), provider
                                .getUrl(), e);
                    }
                }
            }
            // 如果上面的代码执行完, 有三种情况, 一没有对应的实现类map为空, 二只找到一个实现类, 三找到多个实现类.
            // 如果是空, 就使用 SimpleLoggerContextFactory.
            // 另外两种都会调用 treemap.getlastkey(). 只不过多个实现类的情况会生成一条warn级别的日志, 表示找到哪些实现最后使用了哪个.
            if (factories.isEmpty()) {
                LOGGER.error("Log4j2 could not find a logging implementation. "
                        + "Please add log4j-core to the classpath. Using SimpleLogger to log to the console...");
                factory = new SimpleLoggerContextFactory();
            } else if (factories.size() == 1) {
                factory = factories.get(factories.lastKey());
            } else {
                final StringBuilder sb = new StringBuilder("Multiple logging implementations found: \n");
                for (final Map.Entry<Integer, LoggerContextFactory> entry : factories.entrySet()) {
                    sb.append("Factory: ").append(entry.getValue().getClass().getName());
                    sb.append(", Weighting: ").append(entry.getKey()).append('\n');
                }
                factory = factories.get(factories.lastKey());
                sb.append("Using factory: ").append(factory.getClass().getName());
                LOGGER.warn(sb.toString());

            }
        } else {
            LOGGER.error("Log4j2 could not find a logging implementation. "
                    + "Please add log4j-core to the classpath. Using SimpleLogger to log to the console...");
            factory = new SimpleLoggerContextFactory();
        }
    }
}
```

---
获取Logger的逻辑
一共提供了10个方法来获取Logger对象.
主要分为两种, getLogger(Class), getLogger(Class, MessageFactory).

主要研究// 1号类型.
使用 getContext() 获取 LoggerContext

```java
//1
// 用给定的 class 对象的全限定名 返回一个 Logger 对象
public static Logger getLogger(final Class<?> clazz) {
    final Class<?> cls = callerClass(clazz);
    return getContext(cls.getClassLoader(), false).getLogger(toLoggerName(cls));
}
// 返回一个name是调用者的Logger对象
public static Logger getLogger() {
    return getLogger(StackLocatorUtil.getCallerClass(2));
}
public static Logger getLogger(final Object value) {
    return getLogger(value != null ? value.getClass() : StackLocatorUtil.getCallerClass(2));
}
public static Logger getLogger(final String name) {
    return name != null ? getContext(false).getLogger(name) : getLogger(StackLocatorUtil.getCallerClass(2));
}

// 2
// MessageFactory 来创建Logger
public static Logger getLogger(final Class<?> clazz, final MessageFactory messageFactory) {
    final Class<?> cls = callerClass(clazz);
    return getContext(cls.getClassLoader(), false).getLogger(toLoggerName(cls), messageFactory);
}
//
public static Logger getLogger(final MessageFactory messageFactory) {
    return getLogger(StackLocatorUtil.getCallerClass(2), messageFactory);
}

public static Logger getLogger(final Object value, final MessageFactory messageFactory) {
    return getLogger(value != null ? value.getClass() : StackLocatorUtil.getCallerClass(2), messageFactory);
}
public static Logger getLogger(final String name, final MessageFactory messageFactory) {
    return name != null ? getContext(false).getLogger(name, messageFactory) : getLogger(
            StackLocatorUtil.getCallerClass(2), messageFactory);
}
protected static Logger getLogger(final String fqcn, final String name) {
    return factory.getContext(fqcn, null, null, false).getLogger(name);
}
public static getRootLogger() {
    // String ROOT_LOGGER_NAME = Strings.EMPTY;
    return getLogger(ROOT_LOGGER_NAME);
}
```

getContext()

```java
// 获取current LoggerContext. 这个Context可能不是用于为调用类创建Logger的Context(这个Conxtext可能不为Logger对象).
public static LoggerContext getContext() {
    try {
        return factory.getContext(FQCN, null, null, true);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, null, null, true);
    }
}
// 如果为 false 就会为寻找合适的 LoggerContext
public static LoggerContext getContext(final boolean currentContext) {
    // TODO: would it be a terrible idea to try and find the caller ClassLoader here?
    try {
        return factory.getContext(FQCN, null, null, currentContext, null, null);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, null, null, currentContext, null, null);
    }
}

public static LoggerContext getContext(final ClassLoader loader, final boolean currentContext) {
    try {
        return factory.getContext(FQCN, loader, null, currentContext);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, loader, null, currentContext);
    }
}

// externalContext 是指一个外部的Context 比如 ServletContext 会和生成的LoggerContext关联.
public static LoggerContext getContext(final ClassLoader loader, final boolean currentContext,
        final Object externalContext) {
    try {
        return factory.getContext(FQCN, loader, externalContext, currentContext);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, loader, externalContext, currentContext);
    }
}

public static LoggerContext getContext(final ClassLoader loader, final boolean currentContext,
        final URI configLocation) {
    try {
        return factory.getContext(FQCN, loader, null, currentContext, configLocation, null);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, loader, null, currentContext, configLocation,
                null);
    }
}

public static LoggerContext getContext(final ClassLoader loader, final boolean currentContext,
        final Object externalContext, final URI configLocation) {
    try {
        return factory.getContext(FQCN, loader, externalContext, currentContext, configLocation, null);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, loader, externalContext, currentContext,
                configLocation, null);
    }
}

// name 是这个 LoggerContext 的名字
public static LoggerContext getContext(final ClassLoader loader, final boolean currentContext,
        final Object externalContext, final URI configLocation, final String name) {
    try {
        return factory.getContext(FQCN, loader, externalContext, currentContext, configLocation, name);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(FQCN, loader, externalContext, currentContext,
                configLocation, name);
    }
}

// 下面三个方法没有被使用, 大概是给子类使用的
protected static LoggerContext getContext(final String fqcn, final boolean currentContext) {
    try {
        return factory.getContext(fqcn, null, null, currentContext);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(fqcn, null, null, currentContext);
    }
}

protected static LoggerContext getContext(final String fqcn, final ClassLoader loader,
        final boolean currentContext) {
    try {
        return factory.getContext(fqcn, loader, null, currentContext);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(fqcn, loader, null, currentContext);
    }
}
protected static LoggerContext getContext(final String fqcn, final ClassLoader loader,
                                          final boolean currentContext, URI configLocation, String name) {
    try {
        return factory.getContext(fqcn, loader, null, currentContext, configLocation, name);
    } catch (final IllegalStateException ex) {
        LOGGER.warn(ex.getMessage() + " Using SimpleLogger");
        return new SimpleLoggerContextFactory().getContext(fqcn, loader, null, currentContext);
    }
}
```

# LoggerContextFactory

看得出来这个工厂应会负责LoggerContext的创建和缓存工作.

```java
public interface LoggerContextFactory {

    LoggerContext getContext(String fqcn, ClassLoader loader, Object externalContext, boolean currentContext);

    LoggerContext getContext(String fqcn, ClassLoader loader, Object externalContext, boolean currentContext,
                             URI configLocation, String name);
    void removeContext(LoggerContext context);
}
```

# LoggerContext

这个应该负责Logger的创建和缓存, 上面的Factory在创建时会要求提供配置参数. 所以是否是一个Context一个配置呢?

```java
public interface LoggerContext {

    Object getExternalContext();

    ExtendedLogger getLogger(String name);

    ExtendedLogger getLogger(String name, MessageFactory messageFactory);

    boolean hasLogger(String name);

    boolean hasLogger(String name, MessageFactory messageFactory);

    boolean hasLogger(String name, Class<? extends MessageFactory> messageFactoryClass);
}
```

先看api包中提供的简单实现, 内部字段大部分是配置信息.
重点是 LoggerRegistry 字段, 它是存储Logger的地方.
它的结构不是简单的Map\<String, Logger\>,
由于LoggerContext提供了两个getLogger(String name, MessageFactory factory)方法.
所以它内部的存储方式是 Map\<String, Map\<String, Logger>>.
第一个String是传入的 MessageFactory的名字, 然后是根据传入的 name 获取对应的Logger.

```java
public class SimpleLoggerContext implements LoggerContext {


    private final PropertiesUtil props;

    // 一个是否包含日志名的标志
    private final boolean showLogName;

    // 是否展示简单名的标志
    private final boolean showShortName;
    /** Include the current time in the log message */
    private final boolean showDateTime;
    /** Include the ThreadContextMap in the log message */
    private final boolean showContextMap;
    /** The date and time format to use in the log message */
    private final String dateTimeFormat;

    private final Level defaultLevel;

    private final PrintStream stream;

    private final LoggerRegistry<ExtendedLogger> loggerRegistry = new LoggerRegistry<>();
    public SimpleLoggerContext() {
        props = new PropertiesUtil("log4j2.simplelog.properties");

        showContextMap = props.getBooleanProperty(SYSTEM_PREFIX + "showContextMap", false);
        showLogName = props.getBooleanProperty(SYSTEM_PREFIX + "showlogname", false);
        showShortName = props.getBooleanProperty(SYSTEM_PREFIX + "showShortLogname", true);
        showDateTime = props.getBooleanProperty(SYSTEM_PREFIX + "showdatetime", false);
        final String lvl = props.getStringProperty(SYSTEM_PREFIX + "level");
        defaultLevel = Level.toLevel(lvl, Level.ERROR);

        dateTimeFormat = showDateTime ? props.getStringProperty(SimpleLoggerContext.SYSTEM_PREFIX + "dateTimeFormat",
                DEFAULT_DATE_TIME_FORMAT) : null;

        final String fileName = props.getStringProperty(SYSTEM_PREFIX + "logFile", SYSTEM_ERR);
        PrintStream ps;
        if (SYSTEM_ERR.equalsIgnoreCase(fileName)) {
            ps = System.err;
        } else if (SYSTEM_OUT.equalsIgnoreCase(fileName)) {
            ps = System.out;
        } else {
            try {
                final FileOutputStream os = new FileOutputStream(fileName);
                ps = new PrintStream(os);
            } catch (final FileNotFoundException fnfe) {
                ps = System.err;
            }
        }
        this.stream = ps;
    }
}
```
