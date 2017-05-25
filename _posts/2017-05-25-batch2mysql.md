---
layout: post
title:  "批量写入Mysql"
date:   "2017-05-25 16:00:00"
categories: "java mysql batch ringbuffer jdbc"
permalink: /archivers/2017-05-25-batch2mysql
---

Mysql是最常用的一种关系型数据库，随着各种ORM框架的演进，操作数据库也越来越简单。

当你碰到一些极端需求时，还是要在JDBC这个层面上来操作数据库。

今里就介绍一下，在批量写Mysql时碰到的问题。

### 真正的批量写

使用JDBC来完成批量Mysql写入的一般方法是:

```
this.connection.setAutoCommit(false);

this.preparedStatement = this.connection.prepareStatement(this.sql);
for (int i = 0; i < this.schema.size(); i++) {
    this.preparedStatement.setObject(i + 1, data[i], this.schema.get(i));
}
this.preparedStatement.addBatch();

...N次

this.preparedStatement.executeBatch();
this.connection.commit();
```

当你写完这段代码之后，测试性能会发现，并没有什么提升。因为实际上还是一条条发送执行的。

当你为DatasourceUrl添加了参数rewriteBatchedStatements=true，才会真正生效。

具体原因可以自行搜索这个参数，网上有很多文章介绍。

改了参数，性能提升三五倍是很轻松的，具体能提升多少要看实际的情况。

### 选择好数据库的伴侣——数据源连接池

开源的数据源连接池非常多，c3p0、druid、Proxool、BoneCP、HikariCP等等。

比较常用的是阿里开源的Druid，性能不是特别突出，但是综合表现不错。

HikariCP也是性能表现很抢眼的选手。

### 缓冲区

涉及到批量写入数据库就意味着可能和缓冲区打交道了。

攒够一个批量（100条），但是也不能无限期的等，需要一个超时时间（15秒）。

缓冲区可以用框架来实现，比如Disruptor的Ringbuffer无锁队列就可以满足这两个要求。

Ringbuffer的具体使用方法这里就不介绍了，初次上手会有一些难度。

Ringbuffer强制了数据对象的复用，来减少JVM GC的次数，对性能的提升非常有帮助。

### 线程数和批量大小

加大处理数据的线程数也是一个提升处理速度的方法，当然也不能一直加大线程数，

根据测试批量写的线程数不要超过Mysql的Cpu核数+1。

批量条数要根据单行数据的大小决定，一般1000条左右。

### Mysql性能优化

Mysql的性能优化去参考网上的文章，这里就不详细介绍了。

大多是关闭一些log，提高一些buffer，改变filesync的方式等。

表上的索引是很影响写入性能的，尽量保证索引精简。

### JDBC优化

根据前面的优化过后，使用JFR对你的程序做一下热点代码分析，就会发现JDBC的问题了。

如果你仔细阅读过Java的MysqlConnector的代码就会发现，可以改造的地方还是挺多的。

下面举几个例子：

1、批量数据的暂存容器

你提交到preparestatment中的数据，会暂时存到一个ArrayList中。

在大批量数据处理时Arraylist就是一个优化点。跟HashMap一样，它也会涉及到resize的过程。

在你初始化一个preparestatment的时候，根据你的批量大小，对其进行capacity的初始化，会带来不小的性能提升。

```
this.batchArgsField = StatementImpl.class.getDeclaredField("batchedArgs");
this.batchArgsField.setAccessible(true);
...
this.batchArgsField.set((StatementImpl) ((DruidPooledPreparedStatement) this.preparedStatement).getStatement(), new ArrayList<Object>(stepSize));

```

2、日期格式化

一般数据库表都会有一些日期类型的字段，比如创建时间、更新时间等。

Java中的日期类型处理性能一直都不是很高，比如SimpleDateFormat就是一个关键点。

代码详见com.mysql.jdbc.PreparedStatement.class.getDeclaredField("tsdf")，这里就不细说了。

如果你的表字段日期类型比较多，可以考虑替换掉SimpleDateFormat的实现。

比如用Calender的接口，通过字符串拼接的方式完成日期数据的格式化。性能也是会有一定的提升的。

3、失败重试

一个完善的数据处理流程，必然要考虑很多异常场景。比如网络闪断，数据库重启等等。

如果在请求失败后重试呢？一般性的做法就是记录下批量写入的每一个参数，重新写入再执行一边。

前面基于ringbuffer做了数据对象的复用，目的是减少gc，这里为了失败重试，再次引入重复对象是很难接受的。

而且statement.setObject过程就是一个java对象序列化成byte的过程，重复执行也很浪费。

最简单的办法就是利用前面初始化好的batchArgs，实际上他里面就记录了你写入的每一行数据。

只要把它拷贝出来，写入新的preparestatement中，就可以再次执行了。

注意executeBatch()一定不能丢掉

```
List<Object> batchParams = ((JDBC4PreparedStatement) ((DruidPooledPreparedStatement) this.preparedStatement).getStatement()).getBatchedArgs();
Connection tc = this.dataSource.getConnection();
tc.setAutoCommit(false);
PreparedStatement tp = tc.prepareStatement(this.sql);
StatementImpl target = (StatementImpl) ((DruidPooledPreparedStatement) tp).getStatement();
List<Object> newParams = new ArrayList<>(batchParams);
this.batchArgsField.set(target, newParams);
tp.executeBatch();
tc.commit();
tp.close();
tc.close();
```

经过这些优化，我的程序（8G的JVM）可以每秒向Mysql写入十几万行数据（无索引的情况）。

