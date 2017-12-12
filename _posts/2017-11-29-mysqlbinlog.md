---
layout: post
title:  "关于Mysql的Binlog抽取"
date:   "2017-11-29 10:00:00"
categories: "mysql binlog"
permalink: /archivers/2017-11-29-mysqlbinlog
---

这个月换了工作，双十一之前入职了京东大数据平台——实时数据组。

着手关于MysqlBinlog相关系统的重构工作，这次就讲讲Binlog吧。


## Binlog转实时数据流的开源动态

伪装MysqlSlave来获取MysqlBinlog的技术这几年已经非常普及了，很多中等规模的公司都有相关的

基础技术组件，大多数是基于阿里几年前开源的Canal项目，在此基础上进行二次开发，将数据写入Kafka

或者其他MQ系统中去，阿里云的RDS还提供类似的服务，也非常省心。

17年年初的时候我也有意要做类似的项目，所以已经进行了最基础的技术选型和调研。

Java系中最早的binlog开源项目有tungsten-replicator 、open-replicator等，后来都慢慢废弃了。

之后Canal几乎统一了这个领域，但是阿里的这个开源项目最近几年的更新频率很一般。

最近两年比较新的项目有mysql-time-machine，zendesk的maxwell都不错。

shyiko的mysql-binlog-connector-java是一个BinlogClient的超精简内核，也被很多开源项目采用。

## Mysql的Binlog

Mysql的Binlog协议和格式我就不多介绍了，网上可以搜到很多。

值得注意的时候，Mysql一定要配置成Row模式的，而且image需要配置成Full模式的。

我采用shyiko的mysql-binlog-connector-java的方案进行了一下性能测试，可以达到80M/s。

## 数据解析

shyiko的mysql-binlog-connector-java已经将数据解析成Java类型了，接下来只需要针对公司的规范

进行数据的清洗和整理即可，最后转成标准序列化格式，比如AVRO、Pb、Json等。

这中间性能损耗最大的地方就是序列化的开销了，要考虑异步或者多线程流等方式解决。

在开发解析逻辑时碰到了不少细节问题，用shyiko的mysql-binlog-connector-java解析的数据与canal

有一些细小的差别，比如datetime和timestamp字段解析的结果不一样，有可能受时区的影响；

还有像text类型的字段是byte数组，需要自己指定charset转为string；decimal也有差异，需要使用

numberformat进行格式化，group设置为false，并且setMinimumFractionDigits(1)。

## 写Kafka有序

将数据写到Kafka的速度是非常快的，但是因为Binlog的特殊性，需要一些设计，来保证数据的有序。

Kafka的Client会把收到的数据根据meta进行进行重新组织，按照broker的使用情况进行分批发送。

所以我们要根据业务情况指定Kafka的MessageKey，将相同业务含义的数据写到同一个Partition

下，保证有序，乱序会导致最新值被老值覆盖。

在写Kafka的过程中难免会有一些失败的情况发生，错误的重试机制由Kafka来保证，并且要强制设置

ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION = 1，以防止重试时发生数据顺序错误。

## Task的结构

|       P1    |    |    B1   |    |         P2       |    |    B2    |    |      P3       |
|:-----------:|:--:|:-------:|:--:|:----------------:|:--:|:--------:|:--:|:-------------:|
|单线程拉Binlog| > |Ringbuffer| > |单线程转化成AVRO对象| > |Ringbuffer| > |序列化并发送Kafka|
|:-----------:|:--:|:-------:|:--:|:----------------:|:--:|:--------:|:--:|:-------------:|
|             |   |          |    |                  | > |Ringbuffer| > |序列化并发送Kafka|

注释：P一般对应一个线程，B对应一个Disrupter的Ringbuffer。

P1和P2环节的性能都非常好，所以采用的是单线程，阶段并行模式。

P3阶段采用并行模式，P2处理好的数据根据一些key进行选择，路由到N个B2中去，一般可以用库表名称。

压测时，这个Task跑起来之后可以让负载达到400%，基本上把Cpu资源跑满了。

## 持久化位点信息

什么是位点信息？

简单来说就是binlog的filename和offset，为了保证at least once，

我们需要定期记录消费的位置，以便任务重启之后，继续消费，不丢数据（可有少量的重复）。

如果P3阶段是单线程的，那么记录位点非常简单，提交kafka成功后，记录最后一条数据的offset，

可以异步刷到远程存储中去，防止阻塞数据流。可是，现在P3阶段是多线程的，如何记录位点呢？

定期让P2对所有的B2发送一条相同的含有位点信息的心跳包（位点值为P2最后处理的一条数据的位点）

通过这种方式，将各个P3的信息进行更新，每次P3将数据成功发送到Kafka后就将其中的心跳包记录下来，

作为一个可信的回退点，注意这个心跳包一定要在数据流中流转，起到挤压数据的作用，

防止某些P3无数据的情况发生。相关代码参考我的sucker项目，这里就不介绍代码细节了。
