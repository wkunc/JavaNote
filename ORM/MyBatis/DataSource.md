Mybatis 在这里使用了工厂模式
# DataSourceFactory
```java
public interface DataSourceFactory {
    void setPropertiess(Properties props);
    DataSource getDataSource();
}
```
在 UnpooledDataSourceFactory 的构造函数中会直接创建
UnpooledDataSource 对象,并且初始化工厂的 dataSource 字段.

setProperties() 方法会使用Properties中的信息设置 dataSource

properties 中的信息必须由"dirver."开头的key才会被视为数据源的信息
如果 datasource 接口中有对应 key 的 setter 方法就会调用其 setter 方法 
```java
public void setProperties(Properties properties) {
    Properties driverProperties = new Properties();
    MetaObject metaDataSource = SystemMetaObject.forObject(dataSource);
    for (Object key : properties.keySet()) {
        String propertyName = (String)key;
        if (propertyName.startsWith(DRIVE_PROPERTY_PREFIX)) {
            String value = properties.getPropert(propertyName);
            driverProperties.setProperty(propertyName.substring(DRIVE_PROPERTY_PREFIX), value);
        } else if (metaDataSource.hasSetter(propertyName)) {
            String value = (String) properties.get(propertyName);
            Object convertedValue = convertValue(metaDataSource, propertyName, value);
            metaDataSource.setValue(propertyName, convertedValue);
        } else {
            throw new DataSourceException("Unknown DataSource property: " + propertyName);
        }
        if (driverProperties.size() > 0) {
            metaDataSource.setValue("driverProperties", driverProperties);
        }
    }
}
```

# DataSource
```java
public interface DataSource extends CommonDataSource, Wrapper {
    Connection getConnectionno() throws SQLException;
    Connection getConnectionno(String uernmame, String password) throws SQLException;
}
```

```java
private ClassLoader driverClassLoader;
private Properties driverProperties;

//缓存所有已注册的数据库驱动 
private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<String, Driver>();

private String driver;
private String url;
private String username;
private String password;
private Boolean autoCommit;
private Integer defaultTransactionIsolationLevel;

// 将DriverManager中注册的所有的Driver缓存到本地
static{
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
        Driver driver = drivers.nextElement();
        registeredDrivers.put(dirver.getClass().getName(), driver);
    }
}
```

对UnPooledDataSource.getConnection() 方法的调用会传到
UnpooledDataSource.doGetConnection() 方法
最后会调用到熟悉的 DriverManager.getConnection() 方法
```java
    @Override
    public Connection getConnection() throws SQLException {
        return doGetConnection(username, password);
    }
    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return doGetConnection(username, password);
    }
    private Connection doGetConnection(String username, String password) throws SQLException {
        Properties props = new Properties();
        if (driverProperties != null) {
            props.putAll(driverProperties);
        }
        if (username != null) {
            props.setProperty("user", username);
        }
        if (password != null) {
            props.setProperty("password", password);
        }
        return doGetConnection(props);
    }

    private Connection doGetConnection(Properties properties) throws SQLException {
        initializeDriver();
        Connection connection = DriverManager.getConnection(url, properties);
        configureConnection(connection);
        return connection;
    }

    private synchronized void initializeDriver() throws SQLException {
        if (!registeredDrivers.containsKey(driver)) {
            Class<?> driverType;
            try {
                if (driverClassLoader != null) {
                    driverType = Class.forName(driver, true, driverClassLoader);
                } else {
                    driverType = Resources.classForName(driver);
                }
                // DriverManager requires the driver to be loaded via the system ClassLoader.
                // http://www.kfu.com/~nsayer/Java/dyn-jdbc.html
                Driver driverInstance = (Driver)driverType.newInstance();
                DriverManager.registerDriver(new DriverProxy(driverInstance));
                registeredDrivers.put(driver, driverInstance);
            } catch (Exception e) {
                throw new SQLException("Error setting driver on UnpooledDataSource. Cause: " + e);
            }
        }
    }

    private void configureConnection(Connection conn) throws SQLException {
        if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
            conn.setAutoCommit(autoCommit);
        }
        if (defaultTransactionIsolationLevel != null) {
            conn.setTransactionIsolation(defaultTransactionIsolationLevel);
        }
    }

```
----------------
DataSource 池化的意义:
1. 数据库连接的创建非常耗时
2. 数据库连接是有限的

池化之后可以:
1. 在开始时就创建一定数的连接, 这样就避免系统在运行时花费时间在连接的创建上
2. 


首先从池中获得连接, 不再使用连接时将连接还给连接池.
如果连接池中的连接数量达到上限, 且都被占用, 请求进入阻塞队列.
如果连接池中没有空闲连接, 但是连接数量没有达到上限, 则会创建连接
如果空闲连接数量超过上限, 则后续返回的空闲连接直接关闭.

如果总连接数上限过大, 可能因连接数过多导致数据库僵死
如果总连接数上限过小, 则无法发挥数据库的全部性能
如果空闲连接上限过大, 就会浪费系统资源维护这些空闲连接 
如果空闲连接上限过小, 出现瞬间的大量请求时, 系统的快速响应能力就比较弱

所以数据库连接池主要维护这些连接, 然后连接在使用完成后要回到池中. 但是Connection是官方定义的接口
其中只有 close() 方法没有返回池中的方法.
主要思路1. 用动态代理将Connection包装, 将对其close()方法的调用改变

# PooledDataSource
PooledDataSource 基于 UnPooledDataSource, 结合 PoolState, PooledConnection实现了池化.
PooledConnection 实现了 InvocationHandler 接口. (ps: 你懂的, 使用动态代理)
PoolState 则管理空闲连接和活跃连接

-----------------
com.mysql.jdbc.Driver 中有如下代码: 
```java
// 向DriverMananger注册JDBC驱动
static {
    try {
        DriverManager.registerDriver(new Driver());
    } catch (SQlException e) {
        throw new RuntimeException("Can\'t register driver!");
    }
}
```
DriverManager  中定义了 registerDrivers 字段用于记录注册的JDBC驱动
```java
private final static CopyOnWriterArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList();

public static synchronized void registerDriver(java.sql.Driver driver, DriverAction da) throws SQLException {
    if(driver != null) {
        registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
    }else {
        //抛出异常...
    }
}
```