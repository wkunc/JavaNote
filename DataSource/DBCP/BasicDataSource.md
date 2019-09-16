# 
```java

protected DataSource createDataSourcce() throws SQLException {
    // 如果当前DataSource调用了close()方法,
    if (closed) {
        throw new SQLException("Data source is closed");
    }

    // 这里是一个双重检查锁包装了线程安全
    if (dataSource != null) {
        return dataSource;
    }
    synchroized(this) {
        if (dataSource != null) {
            return dataSource;
        }

        jmxRegister();

        // 创建了DriverConnectionFactory, 顾名思义就是使用确定的Driver,创建Connection的工厂
        ConnectionFactory driverConnectionFactory = createConnectionFactory();
        // 下面是基于上面的DriverConnectionFactory创建PoolableConnectionFactory
        boolean success = false;
        PoolableConnectionFactory poolableConnectionFactory;
        try {
            poolableConnectionFactory = createPoolableConnectionFactory(driverConnectionFactory);
            poolableConnectionFactory.setPoolStatements(poolPreparedStatements);
            poolableConnectionFactory.setMaxOpenPrepatedStatements(maxOpenPreparedStatements);
            success = true;
        } catch () {
        } catch () {
        } catch () {
        }

        if (success) {
            createConnectionPool(poolableConnectionFactory);
        }

        DataSource newDataSource;
        success = false;
        try {
            newDataSource = createDataSourceInstance();
            newDataSource.setLogWriter(logWriter);
            success = true;
        } finally {
            if (!success) {
                closeConnectionPool();
            }
        }

        //
        try {
            for (int i = 0; i < initialSize; i++) {
                connectionPool.addObject();
            }
        }

        startPoolMaintenace();
        dataSource = newDataSource;
        return dataSource;
    }

}
```
