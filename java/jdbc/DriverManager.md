# DriverManager

用于管理一组JDBC驱动程序的基本服务.
*DriverManager*是我们使用JDBC的主要核心.

NOTE: 在新的JDBC2.0API中, 提供了javax.sql.DataSource接口.
使用*DataSource*对象是连接到数据原的首选方法.

作为初始化的一部分, DriverManager将尝试加载在 *jdbc.drivers* 系统属性中引用的驱动程序类.
这允许用户自定义其应用程序使用的JDBC驱动程序.
例如, 在 ~/.hotjava/properties 文件中添加:
jdbc.drivers=foo.bah.Driver:wombat.sqlDriver:bad.taste.ourDriver

*DriverManager*在JDBC4.0或者说Java1.6之后会使用 ServiceLoader spi机制自动加载驱动.

主要使用方法:

1. getConnection(String url) 和它的2个重载方法
2. getDriver(String url)
3. getDrivers() 在11后添加了一个 Stream\<Driver> drivers() 的方法.

# init

在DriverManager中的四个方法

1. private static Connection getConnection()
2. public static Stream<Driver> drivers()
2. public static Enumeration<Driver> getDrivers()
3. public static Driver getDriver(String url)

中都有一个 private static void ensureDriversInitialized() 方法调用.
这个方法确保的自动加载 Dirver 机制.

```java
private static void ensureDriversInitialized() {
    if (driversInitialized) {
        return;
    }

    synchronized (lockForInitDrivers) {
        if (driversInitialized) {
            return;
        }
        String drivers;
        try {
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty(JDBC_DRIVERS_PROPERTY);
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }

        // 用 SPI 机制加载Driver
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                try {
                    while (driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch (Throwable t) {
                    // Do nothing
                }
                return null;
            }
        });

        // 加载系统属性 jdbc.drivers 中指定的Driver类
        println("DriverManager.initialize: jdbc.drivers = " + drivers);

        if (drivers != null && !drivers.equals("")) {
            String[] driversList = drivers.split(":");
            println("number of Drivers:" + driversList.length);
            for (String aDriver : driversList) {
                try {
                    println("DriverManager.Initialize: loading " + aDriver);
                    Class.forName(aDriver, true,
                            ClassLoader.getSystemClassLoader());
                } catch (Exception ex) {
                    println("DriverManager.Initialize: load failed: " + ex);
                }
            }
        }

        driversInitialized = true;
        println("JDBC DriverManager initialized");
    }
}
```
