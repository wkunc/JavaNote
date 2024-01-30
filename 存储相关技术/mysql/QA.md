# jdbc 批量插入

## 问题描述
数据库中存储人脸照片, 想在参数中使用 `InputStream`, 而不是`byte[]`.
这样驱动读完流后就可以释放对应的内存, 防止导入时oom.

```java
try (Connection connection = ds.getConnection()) {
    connection.setAutoCommit(false);

    PreparedStatement preparedStatement = connection.prepareStatement("insert into person (id, phone, name, face_pic) values (?, ?, ?, ?) ");
    var p1 = Tuple.of("1", "1", "1", Files.newInputStream(Paths.get("test.jpeg")));
    preparedStatement.setString(1, p1._1);
    preparedStatement.setString(2, p1._2);
    preparedStatement.setString(3, p1._3);
    preparedStatement.setBlob(4, p1._4));
    preparedStatement.addBatch();

    var p2 = Tuple.of("2", "2", "2", Files.newInputStream(Paths.get("test.jpeg")));
    preparedStatement.setString(1, p2._1);
    preparedStatement.setString(2, p2._2);
    preparedStatement.setString(3, p2._3);
    preparedStatement.setBlob(4, p2._4));
    preparedStatement.addBatch();

    int[] ints = preparedStatement.executeBatch();
    System.out.println("成功插入: " + ints.length);

    connection.commit();
} catch (SQLException | IOException e) {
    throw new RuntimeException(e);
}
```
执行报错
>java.lang.NullPointerException
>    at com.mysql.cj.ClientPreparedQuery.computeMaxParameterSetSizeAndBatchSize(ClientPreparedQuery.java:66)
>    at com.mysql.cj.AbstractPreparedQuery.computeBatchSize(AbstractPreparedQuery.java:136)
>    at com.mysql.cj.jdbc.ClientPreparedStatement.executeBatchedInserts(ClientPreparedStatement.java:670)
>    at com.mysql.cj.jdbc.ClientPreparedStatement.executeBatchInternal(ClientPreparedStatement.java:426)
>    at com.mysql.cj.jdbc.StatementImpl.executeBatch(StatementImpl.java:796)

## 原因分析

```java
    @Override
    protected long[] executeBatchInternal() throws SQLException {
        synchronized (checkClosed().getConnectionMutex()) {

            if (this.connection.isReadOnly()) {
                throw new SQLException(Messages.getString("PreparedStatement.25") + Messages.getString("PreparedStatement.26"),
                        MysqlErrorNumbers.SQL_STATE_ILLEGAL_ARGUMENT);
            }

            if (this.query.getBatchedArgs() == null || this.query.getBatchedArgs().size() == 0) {
                return new long[0];
            }

            // we timeout the entire batch, not individual statements
            int batchTimeout = getTimeoutInMillis();
            setTimeoutInMillis(0);

            resetCancelledState();

            try {
                statementBegins();

                clearWarnings();

                if (!this.batchHasPlainStatements && this.rewriteBatchedStatements.getValue()) {
                    // 执行批量删除插入 
                    if (((PreparedQuery<?>) this.query).getParseInfo().canRewriteAsMultiValueInsertAtSqlLevel()) {
                        return executeBatchedInserts(batchTimeout);
                    }

                    if (!this.batchHasPlainStatements && this.query.getBatchedArgs() != null
                            && this.query.getBatchedArgs().size() > 3 /* cost of option setting rt-wise */) {
                        return executePreparedBatchAsMultiStatement(batchTimeout);
                    }
                }

                return executeBatchSerially(batchTimeout);
            } finally {
                this.query.getStatementExecuting().set(false);

                clearBatch();
            }
        }
    }

    // Rewrites the already prepared statement into a multi-value insert statement of 'statementsPerBatch' values and executes the entire batch using this new statement.
    // 我理解的是把多条 insert into person (id, name, face) values (?, ?, ?)
    // 合并为一条 insert into person (id, name, face) values (?, ?, ?), (?, ?, ?),(?, ?, ?)
    protected long[] executeBatchedInserts(int batchTimeout) throws SQLException {
        synchronized (checkClosed().getConnectionMutex()) {
            String valuesClause = ((PreparedQuery<?>) this.query).getParseInfo().getValuesClause();

            JdbcConnection locallyScopedConn = this.connection;

            if (valuesClause == null) {
                return executeBatchSerially(batchTimeout);
            }

            int numBatchedArgs = this.query.getBatchedArgs().size();

            if (this.retrieveGeneratedKeys) {
                this.batchedGeneratedKeys = new ArrayList<>(numBatchedArgs);
            }

            // 计算每批的处理的大小
            int numValuesPerBatch = ((PreparedQuery<?>) this.query).computeBatchSize(numBatchedArgs);

            if (numBatchedArgs < numValuesPerBatch) {
                numValuesPerBatch = numBatchedArgs;
            }

            JdbcPreparedStatement batchedStatement = null;

            int batchedParamIndex = 1;
            long updateCountRunningTotal = 0;
            int numberToExecuteAsMultiValue = 0;
            int batchCounter = 0;
            CancelQueryTask timeoutTask = null;
            SQLException sqlEx = null;

            long[] updateCounts = new long[numBatchedArgs];

            try {
                try {
                    batchedStatement = /* FIXME -if we ever care about folks proxying our JdbcConnection */
                            prepareBatchedInsertSQL(locallyScopedConn, numValuesPerBatch);

                    timeoutTask = startQueryTimer(batchedStatement, batchTimeout);

                    numberToExecuteAsMultiValue = numBatchedArgs < numValuesPerBatch ? numBatchedArgs : numBatchedArgs / numValuesPerBatch;

                    int numberArgsToExecute = numberToExecuteAsMultiValue * numValuesPerBatch;

                    for (int i = 0; i < numberArgsToExecute; i++) {
                        if (i != 0 && i % numValuesPerBatch == 0) {
                            try {
                                updateCountRunningTotal += batchedStatement.executeLargeUpdate();
                            } catch (SQLException ex) {
                                sqlEx = handleExceptionForBatch(batchCounter - 1, numValuesPerBatch, updateCounts, ex);
                            }

                            getBatchedGeneratedKeys(batchedStatement);
                            batchedStatement.clearParameters();
                            batchedParamIndex = 1;

                        }

                        batchedParamIndex = setOneBatchedParameterSet(batchedStatement, batchedParamIndex, this.query.getBatchedArgs().get(batchCounter++));
                    }

                    try {
                        updateCountRunningTotal += batchedStatement.executeLargeUpdate();
                    } catch (SQLException ex) {
                        sqlEx = handleExceptionForBatch(batchCounter - 1, numValuesPerBatch, updateCounts, ex);
                    }

                    getBatchedGeneratedKeys(batchedStatement);

                    numValuesPerBatch = numBatchedArgs - batchCounter;
                } finally {
                    if (batchedStatement != null) {
                        batchedStatement.close();
                        batchedStatement = null;
                    }
                }

                try {
                    if (numValuesPerBatch > 0) {
                        batchedStatement = prepareBatchedInsertSQL(locallyScopedConn, numValuesPerBatch);

                        if (timeoutTask != null) {
                            timeoutTask.setQueryToCancel(batchedStatement);
                        }

                        batchedParamIndex = 1;

                        while (batchCounter < numBatchedArgs) {
                            batchedParamIndex = setOneBatchedParameterSet(batchedStatement, batchedParamIndex, this.query.getBatchedArgs().get(batchCounter++));
                        }

                        try {
                            updateCountRunningTotal += batchedStatement.executeLargeUpdate();
                        } catch (SQLException ex) {
                            sqlEx = handleExceptionForBatch(batchCounter - 1, numValuesPerBatch, updateCounts, ex);
                        }

                        getBatchedGeneratedKeys(batchedStatement);
                    }

                    if (sqlEx != null) {
                        throw SQLError.createBatchUpdateException(sqlEx, updateCounts, this.exceptionInterceptor);
                    }

                    if (numBatchedArgs > 1) {
                        long updCount = updateCountRunningTotal > 0 ? java.sql.Statement.SUCCESS_NO_INFO : 0;
                        for (int j = 0; j < numBatchedArgs; j++) {
                            updateCounts[j] = updCount;
                        }
                    } else {
                        updateCounts[0] = updateCountRunningTotal;
                    }
                    return updateCounts;
                } finally {
                    if (batchedStatement != null) {
                        batchedStatement.close();
                    }
                }
            } finally {
                stopQueryTimer(timeoutTask, false, false);
                resetCancelledState();
            }
        }
    }

    // 协议自然会有最大的包大小, 所有需要计算出合适的批次数量. 来合并insert.
    public int computeBatchSize(int numBatchedArgs) {
        long[] combinedValues = computeMaxParameterSetSizeAndBatchSize(numBatchedArgs);

        // 参数集中的最大的参数列表值
        long maxSizeOfParameterSet = combinedValues[0];
        // 总字节数
        long sizeOfEntireBatch = combinedValues[1];

        // 如果小于 maxAllowedPacket, 说明一次就可以放下全部参数集, 返回数量
        if (sizeOfEntireBatch < this.maxAllowedPacket.getValue() - this.originalSql.length()) {
            return numBatchedArgs;
        }

        // 否则按照最大参数计算, maxAllowedPacket 内可以放下多少条
        return (int) Math.max(1, (this.maxAllowedPacket.getValue() - this.originalSql.length()) / maxSizeOfParameterSet);
    }

    /**
     * Computes the maximum parameter set size, and entire batch size given
     * the number of arguments in the batch.
     * 返回最大的参数集大小和参数的总大小 
     */
    @Override
    protected long[] computeMaxParameterSetSizeAndBatchSize(int numBatchedArgs) {
        long sizeOfEntireBatch = 0;
        long maxSizeOfParameterSet = 0;

        for (int i = 0; i < numBatchedArgs; i++) {
            ClientPreparedQueryBindings qBindings = (ClientPreparedQueryBindings) this.batchedArgs.get(i);

            BindValue[] bindValues = qBindings.getBindValues();

            long sizeOfParameterSet = 0;

            for (int j = 0; j < bindValues.length; j++) {
                if (!bindValues[j].isNull()) {

                    if (bindValues[j].isStream()) {
                        long streamLength = bindValues[j].getStreamLength();

                        if (streamLength != -1) {
                            sizeOfParameterSet += streamLength * 2; // for safety in escaping
                        } else {
                            int paramLength = qBindings.getBindValues()[j].getByteValue().length;
                            sizeOfParameterSet += paramLength;
                        }
                    } else {
                        sizeOfParameterSet += qBindings.getBindValues()[j].getByteValue().length;
                    }
                } else {
                    sizeOfParameterSet += 4; // for NULL literal in SQL 
                }
            }

            //
            // Account for static part of values clause
            // This is a little naive, because the ?s will be replaced but it gives us some padding, and is less housekeeping to ignore them. We're looking
            // for a "fuzzy" value here anyway
            //

            if (this.parseInfo.getValuesClause() != null) {
                sizeOfParameterSet += this.parseInfo.getValuesClause().length() + 1;
            } else {
                sizeOfParameterSet += this.originalSql.length() + 1;
            }

            sizeOfEntireBatch += sizeOfParameterSet;

            if (sizeOfParameterSet > maxSizeOfParameterSet) {
                maxSizeOfParameterSet = sizeOfParameterSet;
            }
        }

        return new long[] { maxSizeOfParameterSet, sizeOfEntireBatch };
    }
```
原因就是重写为多值插入语句时, 需要知道每批参数大小来确保生成的语句不会超过server设置的 `maxAllowedPacket` 值.

## 解决办法
使用可以指定`stream`长度的方法, 明确告诉`driver`这个流有多大.

```java
void setBlob(int parameterIndex, InputStream inputStream, long length)
void setBinaryStream(int parameterIndex, java.io.InputStream x,
                     int length) throws SQLException;

```

## rewriteBatchedStatements属性
![官方说明](https://dev.mysql.com/doc/connector-j/en/connector-j-connp-props-performance-extensions.html#cj-conn-prop_rewriteBatchedStatements)
>
> Notice that this might allow SQL injection when using plain statements and the provided input is not properly sanitized. Also notice that for prepared statements, if the stream length is not specified when using 'PreparedStatement.set*Stream()', the driver would not be able to determine the optimum number of parameters per batch and might return an error saying that the resultant packet is too large.

## 特殊情况
目前发现 8.0.17 版本mysql区域在进行insert sql解析时. 如果是用 `value` 关键字会认为无法进行sql层面的重写. 从而回退到单条执行.
```sql
insert into xxx (colum1, colum2) value (?,?)
```
较新版本驱动不会区分 `values` `value`.

