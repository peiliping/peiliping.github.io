---
layout: post
title:  "MysqlBinlog中的事务"
date:   "2018-03-22 10:00:00"
---

binlog抽取的项目测试了一段时间，一直都很顺利，最近扩大了测试的规模，发现了一些细节问题。

## Transaction中的TableId

在binlog中有一个TableId字段，主要是用来关联TableMap和之后的RowEvent的。如果你在搜索引擎

中搜索关于TableId的信息，会找到很多于此有关的Bug和故障，比如超过Int的最大值。这些细节就

不再这里赘述了，总之在一个binlog局部片段中TableMap和RowEvent需要TableId进行关联。

最初我们的代码是这样处理的，当遇到TableMapEvent就把它赋值给一个成员变量tablemap，

遇到XIDEvent就让tablemap=null。这样处理其实是假定单一事务内只有一个tablemap，

但这是不正确的。在一个事务里有可能碰到多个tablemap：


````

tablemap(id=7000)

tablemap(id=7002)

rowdata(id=7000)

rowdata(id=7002)

xid

````

这样就会导致第二条rowdata匹配了错误的tablemap，也就无法对数据进行有效的解析了。

## 解决办法

为了解决这个问题我们翻看了canal和其他几个binlog处理的开源项目代码，大体解决方案为两类。

1、用一个Hashmap把TableId和Tablemap缓存下来，当事务结束后，将mapclear掉。

2、用一个全局的map将TableId和TableMapCache下来。

方案二显然是有问题的，因为TableId并非有限范围，随着时间的累计会导致OOM。

方案一是一个有效方案但是效率并不是特别高。

在方案一的基础上我们做了一些改进：

1、保留我们的tablemap成员对象，用来cache一个事务中的第一个tablemap，这个其实非常必要，

这个tablemap的binlog位置会成为后面事务中其他数据的offset，可以作为有效的回退位点。

2、再增加一个hashmap成员变量cache，当tablemap!=null并且当前event的type为tablemap时，

判断TableId是不是相等的，如果不相等，将tablemap缓存到cache中，key为tableid。

3、当event类型为row时，判断tableid与tablemap的tableid是不是相等，如果相等则用tablemap。

如果不相等，就从cache中查找一个tablemap。

4、当event类型为xid时，tablemap=null，cache.clear()

这样在绝大多数情况下，不会启动hashmap，几乎不会产生额外的开销。


```

 	EventType eventType = event.getHeader().getEventType();
        long dataTableId;
        switch (eventType) {
            case PRE_GA_WRITE_ROWS:
            case WRITE_ROWS:
            case EXT_WRITE_ROWS:
                dataTableId = ((WriteRowsEventData) event.getData()).getTableId();
                break;
            case PRE_GA_UPDATE_ROWS:
            case UPDATE_ROWS:
            case EXT_UPDATE_ROWS:
                dataTableId = ((UpdateRowsEventData) event.getData()).getTableId();
                break;
            case PRE_GA_DELETE_ROWS:
            case DELETE_ROWS:
            case EXT_DELETE_ROWS:
                dataTableId = ((DeleteRowsEventData) event.getData()).getTableId();
                break;
            case TABLE_MAP:
                dataTableId = ((TableMapEventData) event.getData()).getTableId();
                if (this.tableMapEvent == null) {
                    this.tableMapEvent = event;
                } else if (dataTableId != ((TableMapEventData) this.tableMapEvent.getData()).getTableId()) {
                    this.cacheTableMapEvent4Transaction.put(dataTableId, event);
                }
                this.binlogFileName = this.mysqlClient.getBinlogFilename();
                this.binlogPosition = ((EventHeaderV4) event.getHeader()).getPosition();
                getCounter().notRowSum.incrementAndGet();
                return;
            case XID:
                this.tableMapEvent = null;
                this.cacheTableMapEvent4Transaction.clear();
                getCounter().notRowSum.incrementAndGet();
                return;
            default:
                getCounter().notRowSum.incrementAndGet();
                return;
        }

        if (((TableMapEventData) this.tableMapEvent.getData()).getTableId() == dataTableId) {
            container.setPre(Validate.notNull(this.tableMapEvent));
        } else {
            Event e = this.cacheTableMapEvent4Transaction.get(dataTableId);
            container.setPre(Validate.notNull(e));
        }

        container.setCur(event);


```
