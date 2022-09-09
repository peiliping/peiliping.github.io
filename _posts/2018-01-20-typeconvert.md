---
layout: post
title:  "类型转换"
date:   "2018-01-20 10:00:00"
---

之前几个月一直在忙一个binlog抽取的项目，将mysqlbinlog拉出来，写入kafka，

之后消费kafka中的数据写到HDFS，文件格式为orcfile+snappy。

## 背景

上一篇blog讲过kafka中的数据是以avro为载体的，其中的数据字段存在一个map中，

map的key和value都是charsequence，也就是说我们的数据在经过avro之后失去了类型信息。

我们的目标端是写入hdfs上的orcFile，如果是写sequencefile，基本上我们就不关心类型了。

阿里早期的数据仓库中，几乎所有的字段类型都是string的，这样做显然会有空间的浪费，

但也比较方便，不容易出错，方便管理。在前一家公司里，我负责的离线数据都是以parquet为主的，

orc相比parquet查询的性能会更快一些。

## 类型转换

orc是有字段类型概念的，那么我们如何将string转成具体的类型呢？

首先我们要知道原始类型（mysql中的类型），还要知道hive表中的字段类型（orc类型）。

知道了这两端的类型，我们就有可能完成这个工作了。最简单的处理办法就是笛卡尔积。

把每种组合的处理函数写好，进行配置就可以了。这样做的缺点非常明显，就是工作量大。

如果有一天我们不再使用orc格式的类型，换成parquet或者其他的，那么还需要大量的重复工作。

于是解决这个问题的关键是降低耦合度，降低复杂度。

之前在阅读阿里开源的datax的时候看到过一个类似问题的解决方案，引入状态机。

mysql的常见字段类型大概不到20种，为每种类型创建一个type类，并提供转成其他类型的方法。


```
public interface JavaType {

    Boolean toBoolean(String value);

    Integer toInt(String value);

    Long toLong(String value);

    Date toDate(String value);

    Float toFloat(String value);

    Double toDouble(String value);

    default String toString(String value) {
        return value;
    }

    default void unsupport() {
        throw new RuntimeException("type error");
    }

}

```

碰到把boolean转date显然是不可能实现的，那么就直接unsupport好了。

接下来解决orc端的问题

```
 TINYINT(new C_LongColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createByte();
        }
    }, SMALLINT(new C_LongColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createShort();
        }
    }, INT(new C_LongColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createInt();
        }
    }, BIGINT(new C_LongColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createLong();
        }
    }, BOOLEAN(new C_BooleanColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createBoolean();
        }
    }, FLOAT(new C_DoubleColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createFloat();
        }
    }, DOUBLE(new C_DoubleColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createDouble();
        }
    }, DECIMAL(new C_DecimalColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createDecimal();
        }
    }, STRING(new C_BytesColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createString();
        }
    }, BINARY(new C_BytesColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createBinary();
        }
    }, CHAR(new C_BytesColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createChar();
        }
    }, VARCHAR(new C_BytesColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createVarchar();
        }
    }, TIMESTAMP(new C_TimeStampColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createTimestamp();
        }
    }, DATE(new C_DateColumnVector()) {
        @Override
        public TypeDescription toOrcTypeDescption() {
            return TypeDescription.createDate();
        }
    };

    private Convert convert;

    OrcTypeEnum(Convert convert) {
        this.convert = convert;
    }

    public static OrcTypeEnum findType(String type) {
        return OrcTypeEnum.valueOf(type.toUpperCase());
    }

    public abstract TypeDescription toOrcTypeDescption();

    public void setValue(ColumnVector vector, int row, String value, JavaType typeConvert) {
        convert.eval(vector, row, value, typeConvert);
    }
```

为orc的每种类型创建一个convert，大概十来种的样子。

通过这种方式，我们不仅解决了正常的类型转化需求，还能够天然支持date to long 和long to date这样的复杂需求，

极大的提高了工具的灵活度。

## 函数化

之前datax的方案是引入一个实体类对原始类型的数据进行包装，如果每一个字段都经过一次包装会严重增加体积，

ygc的频率会提高，所以在我们的方案中是函数化的，相关类型进行编号，性能也有一些提升。

```
    public void writeData(List<String> result, List<ColumnInfo> columnInfos, List<JavaType> javaTypeList) {
        int row = this.batch.size++;
        for (int i = 0; i < result.size(); i++) {
            ColumnInfo col = columnInfos.get(i);
            col.getHiveTypeEnum().setValue(batch.cols[i], row, result.get(i), javaTypeList.get(col.getJavaTypeFunctionId()));
        }
    }
```

