---
layout: post
title:  "批量写入Mysql"
date:   "2017-05-25 16:00:00"
categories: "java mysql batch ringbuffer jdbc"
permalink: /archivers/2017-05-25-batch2mysql
---

Mysql是最常用的一种关系型数据库，随着各种ORM框架的演进，操作数据库也变得越来越简单。

但是，当你碰到一些极端需求时，还是要回到JDBC这个层面上来操作。

上篇博客介绍了Meepo这个数据迁移工具，其中就涉及到大量的Mysql写入操作。

下面介绍一下，我在批量写Mysql时碰到的一些问题。

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

当你写完这段代码，测试性能就会发现，与单条Insert相比，并没有什么提升。

因为，JDBC实际上仍是以行为单位，发送和执行写入操作的，Batch是假的。

当你为Datasource的Url添加了参数rewriteBatchedStatements=true，才会真正批量执行。

具体原因可以自行搜索这个参数，网上有很多文章介绍，性能可以提升三五倍。

### 选择好数据库的伴侣——数据源连接池

开源的数据源连接池非常多，比如C3P0、Druid、Proxool、BoneCP、HikariCP...

比较常用的是阿里开源的Druid，综合表现不错。HikariCP也是不错的选择。

HikariCP和Druid的作者在Github上有一段关于性能测试的争论也可以看看。

### 缓冲区

涉及到批量操作，就意味着要和缓冲区打交道了，你可能会想到数组、队列、栈等等。

一般来说批量写入的缓冲区有这么几个要求：

1、容量空间可控，比如1024条，写满的时候可以阻塞住。

2、有超时机制，不要无限期等待。

3、性能好，最好是无锁的。

Disruptor的Ringbuffer可以满足上面的这些需求，只是初次上手会有一些难度。

Ringbuffer强制了数据对象的复用，可以减少JVM GC的次数，对性能的提升非常有帮助。

Ringbuffer的具体使用方法这里就不介绍了，网上有很多文章和例子。

### 线程数和批量大小

适当的加大并行度和批量大小，是可以提高消费速度的。

根据测试批量写的线程数不要超过Mysql的Cpu核数+1。

批量的条数由单行数据大小决定，一般1000条左右。

### Mysql性能优化

Mysql性能优化，参考网上的文章，这里就不详细介绍了。

大多是关闭一些log，提高一些buffer，改变filesync的方式等。

我们是使用阿里云的RDS进行的测试，几乎不用什么调整，性能就很不错了。

需要注意表上的索引个数，索引是很影响写入性能的，尽量保证索引精简。

### JDBC优化

经过前面的优化后，再使用JFR对你的程序做一下热点代码的分析，就会发现集中在JDBC的代码上了。

仔细阅读MysqlConnector的代码就会发现，可以改造的地方还是挺多的。

下面举几个例子：

1、批量数据的暂存容器

提交到Preparestatment中的数据，会暂时存到一个ArrayList中。

ArrayList跟HashMap一样，随着数据被多次add，就会触发resize和array copy。

通过Arrays类的copyOf方法对原数组进行拷贝，长度为原数组的1.5倍+1。

在你初始化一个Preparestatment的时候，根据你的批量大小，对其进行capacity的初始化，会带来不小的性能提升。

```
this.batchArgsField = StatementImpl.class.getDeclaredField("batchedArgs");
this.batchArgsField.setAccessible(true);
...
this.batchArgsField.set((StatementImpl) ((DruidPooledPreparedStatement) this.preparedStatement).getStatement(), new ArrayList<Object>(stepSize));

```

2、日期格式化

一般数据库表都会有一些日期类型的字段，比如创建时间、更新时间等。

Java中的日期类型格式化处理性能一般，比如SimpleDateFormat就是一个关键点。

代码com.mysql.jdbc.PreparedStatement中的tsdf，就是用来格式化日期类型的。

假如你的表中日期类型字段比较多，可以考虑替换掉SimpleDateFormat的实现。

Mysql中的日期类型格式是比较固定的，并不需要SimpleDateFormat那么复杂的实现，

可以使用Calender的接口，通过字符串拼接的方式完成日期数据的格式化。

性能也是会有一定的提升。

```
public class DateFormatter extends SimpleDateFormat {

    public DateFormatter(String pattern) {
        super("", Locale.US);
    }

    @Override public StringBuffer format(Date date, StringBuffer toAppendTo, FieldPosition pos) {
        super.calendar.setTime(date);
        toAppendTo.append("'");
        toAppendTo.append(super.calendar.get(1));
        toAppendTo.append("-");
        if (super.calendar.get(2) < 9) {
            toAppendTo.append(0);
        }
        toAppendTo.append(super.calendar.get(2) + 1);
        toAppendTo.append("-");
        if (super.calendar.get(5) < 10) {
            toAppendTo.append(0);
        }
        toAppendTo.append(super.calendar.get(5));
        toAppendTo.append(" ");
        if (super.calendar.get(11) < 10) {
            toAppendTo.append(0);
        }
        toAppendTo.append(super.calendar.get(11));
        toAppendTo.append(":");
        if (super.calendar.get(12) < 10) {
            toAppendTo.append(0);
        }
        toAppendTo.append(super.calendar.get(12));
        toAppendTo.append(":");
        if (super.calendar.get(13) < 10) {
            toAppendTo.append(0);
        }
        toAppendTo.append(super.calendar.get(13));
        return toAppendTo;
    }
}
```

3、异常处理

一个完善的数据处理流程，必然要考虑异常情况，比如网络闪断，数据库重启等等。

如何在请求失败后重试呢？一般的做法就是记录下批量写入的每一个数据，异常时重新写入一遍。

前面基于Ringbuffer做了数据对象的复用，为了处理异常再次引入重复对象是很难接受的。

最简单的办法就是利用前面初始化好的batchArgs，实际上它里面就记录了你写入的每一行数据。

当出现异常时，只要把它拷贝出来，写入新的Preparestatement中，就可以再次执行了。

注意executeBatch一定不能丢掉。

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

### SQL语法

批量写入的SQL语法也不止一种，比较常见的有：

1、INSERT INTO xxxx (...) VALUES (...)

2、INSERT IGNORE INTO xxxx (...) VALUES (...)

3、REPLACE INTO xxxx (...) VALUES (...)

当主键冲突的时候如何处理，可以根据情况选择合适的SQL模式。

### 参数

最后附上我所使用的URL参数

```
rewriteBatchedStatements=true
useUnicode=true
characterEncoding=UTF-8
useSSL=false
verifyServerCertificate=false
failOverReadOnly=false
autoReconnect=true
autoReconnectForPools=true
```

经过这些优化，Meepo可以每秒从Mysql中读取十几万的数据，并写入到一个无索引的新表中。

8G的JVM，YGC每8-10秒发生一次，Mysql的Cpu会被压满。
