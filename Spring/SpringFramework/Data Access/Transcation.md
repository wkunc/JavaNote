# PlatformTranscationManager
```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

```java
public interface TransactionStatus extends SavepointManager, Flushable {
    boolean isNewTransaction();
    boolean hasSavepoint();
    void setRollbackOnly();
    boolean isRollbackOnly();
    @Override
    void flush();
    boolean isCompleted();
}
```
# 声明式事务


# Programmatic Transaction Management(编程使用事务)
1. Transaction Template
2. PlatformTransactionManager


