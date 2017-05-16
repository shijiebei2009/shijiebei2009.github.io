title: "MySQL 拉取海量数据报 OutOfMemoryError"
date: 2017-05-12 22:40:37
tags: [MySQL]
categories: Database

---

在用最基本的`JDBC`拉取数据的时候，由于拉取的是海量数据，所以程序跑了一段时间之后报`java.lang.OutOfMemoryError: Java heap space`，这个错误很简单，也很好解决，网上一搜一大把，只需要设置`ResultSet`获取数据模式为`row-by-row`，但是总结多数的解决方案是如下两种：
① 以PreparedStatement为例，需要设置四个参数
```java
preparedStatement = connection.prepareStatement(formatSql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
preparedStatement.setFetchSize(Integer.MIN_VALUE);
preparedStatement.setFetchDirection(ResultSet.FETCH_REVERSE);
```

② 同样以PreparedStatement为例，需要设置三个参数
```java
preparedStatement = connection.prepareStatement(formatSql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
preparedStatement.setFetchSize(Integer.MIN_VALUE);
```

这种解决方案是可以的，那么本文还有无存在的必要呢？当然有。这两种方案基本上都是参看MySQL官方说明来解决的，具体链接[点我](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-implementation-notes.html)，内容摘录如下
>By default, ResultSets are completely retrieved and stored in memory. In most cases this is the most efficient way to operate and, due to the design of the MySQL network protocol, is easier to implement. If you are working with ResultSets that have a large number of rows or large values and cannot allocate heap space in your JVM for the memory required, you can tell the driver to stream the results back one row at a time.

>To enable this functionality, create a Statement instance in the following manner:

>stmt = conn.createStatement(java.sql.ResultSet.TYPE_FORWARD_ONLY, java.sql.ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(Integer.MIN_VALUE);

>The combination of a forward-only, read-only result set, with a fetch size of Integer.MIN_VALUE serves as a signal to the driver to stream result sets row-by-row. After this, any result sets created with the statement will be retrieved row-by-row.

>There are some caveats with this approach. You must read all of the rows in the result set (or close it) before you can issue any other queries on the connection, or an exception will be thrown.

>The earliest the locks these statements hold can be released (whether they be MyISAM table-level locks or row-level locks in some other storage engine such as InnoDB) is when the statement completes.

>If the statement is within scope of a transaction, then locks are released when the transaction completes (which implies that the statement needs to complete first). As with most other databases, statements are not complete until all the results pending on the statement are read or the active result set for the statement is closed.

>Therefore, if using streaming results, process them as quickly as possible if you want to maintain concurrent access to the tables referenced by the statement producing the result set.

但是我想说，其实一个参数就足矣。只需要设置`fetch size`为`Integer.MIN_VALUE`即可。代码如下：
```java
preparedStatement = connection.prepareStatement(formatSql);
preparedStatement.setFetchSize(Integer.MIN_VALUE);
```

这样为什么可以呢？我们来看源码，点开prepareStatement的具体实现。

```java
package com.mysql.jdbc;
public class ConnectionImpl extends ConnectionPropertiesImpl implements MySQLConnection {
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        return this.prepareStatement(sql, 1003, 1007);
    }
}
```
可以看到即使你调用的是`prepareStatement(formatSql)`，但是在实现中调用的是`prepareStatement(sql, 1003, 1007)`，而`ResultSet.TYPE_FORWARD_ONLY = 1003`，`ResultSet.CONCUR_READ_ONLY = 1007`，所以不需要在调用的时候传递`TYPE_FORWARD_ONLY `和`CONCUR_READ_ONLY`。

再来看`setFetchDirection`的具体实现类
```java
package com.mysql.jdbc;
public class StatementImpl implements Statement {
    public void setFetchDirection(int direction) throws SQLException {
        switch(direction) {
        case 1000:
        case 1001:
        case 1002:
            return;
        default:
            throw SQLError.createSQLException(Messages.getString("Statement.5"), "S1009", this.getExceptionInterceptor());
        }
    }
}
```
可知，在实现中，当`direction`值是1000、1001和1002时，其处理逻辑是一样的，那么这些值表示什么意思呢？在`ResultSet`类中可以查到
```java
int FETCH_FORWARD = 1000;
int FETCH_REVERSE = 1001;
int FETCH_UNKNOWN = 1002;
```
所以再调用`preparedStatement.setFetchDirection(ResultSet.FETCH_REVERSE);`这一句其实完全没必要，因为不论你传递的是哪个值，其结果都是相同的，所以说，使用流式结果集获取海量数据一个参数足矣，不要迷信网上二手信息，同样不要迷信官网，只有源码最靠谱。

如果想继续深究，可以查看MySQL判断是否开启流式结果集的方法，实现如下，判断逻辑很简单
```java
package com.mysql.jdbc;
public class StatementImpl implements Statement {
    protected boolean createStreamingResultSet() {
        return this.resultSetType == 1003 && this.resultSetConcurrency == 1007 && this.fetchSize == -2147483648;
    }
}
```