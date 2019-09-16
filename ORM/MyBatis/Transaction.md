# TranSaction
首先JDBC没有提供事务抽象, rollback(), commit()等方法都是在Connection中的
所以事务抽象基本就是正确调用 Connection 上的方法
```java
public interface Transaction {

  Connection getConnection() throws SQLException;

  void commit() throws SQLException;

  void rollback() throws SQLException;

  void close() throws SQLException;

  Integer getTimeout() throws SQLException;
  
}
```

```java
public class JdbcTransactionFactory implements TransactionFactory {

    @Override
    public void setProperties(Properties props) {
    }

    @Override
    public Transaction newTransaction(Connection conn) {
        return new JdbcTransaction(conn);
    }
    @Override
    // 个人喜欢这个方法, 这样就是事务拥有外界给它的数据源
    // 它获取连接设置隔离等级, 是否自动提交等等
    public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
        return new JdbcTransaction(ds, level, autoCommit);
    }
}
```
简单分析下MyBatis的 JdbcTransaction
```java
protected Connection connection;
protected DataSource dataSource;
protected TransactionIsolationLevel level;
protected boolean autoCommit;

// 在创建的过程中就会初始化对应变量, 除了 Connection 变量会延迟到调用 getConnection() 方法
public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommit = desiredAutoCommit;
}
```
