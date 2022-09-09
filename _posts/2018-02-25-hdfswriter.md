---
layout: post
title:  "分发log数据到HDFS"
date:   "2018-02-25 10:00:00"
---

将Kafka中的数据分发到HDFS上，提到这个需求首先想到的就是Flume。

之前我也在Flume上做了很多改进来提高性能，因为整体框架的约束，只是修改一些皮毛。

最近正好做了一个log分发的项目，作用和Flume非常相似，初步性能测试比Flume快很多。

每秒钟可以从Kafka拉取300M的数据，写HDFS也大概了每秒100M。

## 切换用户

HDFS一般都有权限管理，最常见的就是用用户名，假设写HDFS的程序所在的linux用户叫flume，

那么HDFS上的文件用户名就是flume。为了方便管理我们的程序肯定是统一使用一个用户来启动的，

这就需要切换  System.setProperty("HADOOP_USER_NAME", this.userName);

在适当的时候执行以上语句，就可以达到切换用户的效果，当然还有其他办法,就不一一介绍了。

## Codec

绝大多数公司写入HDFS上的数据都会经过压缩，常见的如snappy，lzo，gzip等等，

为了灵活方便，程序里通过配置来选择压缩工具，类似SPI的一种机制。

其中值得注意的是lzo和lzop需要单独配置一下。

`````

List<Class<? extends CompressionCodec>> codecs = CompressionCodecFactory.getCodecClasses(conf);
codecs.add(LzoCodec.class);
codecs.add(LzopCodec.class);

`````

## sync

HDFS上一般一个block大小是128M左右，所以我们在写文件的过程中一定会sync一下，

HDFS的FileSystem API中关于sync有好多种，我们参照Flume选择了hflush，性能还是非常不错的。

## 分区

之前介绍过Flume的路径渲染极大影响了它的吞吐能力，在我们的程序中也极力避免复杂的路径渲染，

比如单sink同时只能写一个文件等，但是分区路径的渲染还是避免不掉的，常见的分区有小时分区和天分区。


```
public enum PartitionType {

    DAY {
        @Override
        public PartitionRender newInstance() {
            return new DayPartitionRender();
        }
    }, HOUR {
        @Override
        public PartitionRender newInstance() {
            return new DayHourPartitionRender();
        }
    };

    public abstract PartitionRender newInstance();

    public abstract class PartitionRender {

        public abstract void render(StringBuilder base);

    }

    public class DayPartitionRender extends PartitionRender {

        private SimpleDateFormat simpleDateFormat = new SimpleDateFormat("/'dt'=yyyy-MM-dd/HH-mm-");

        @Override
        public void render(StringBuilder base) {
            base.append(simpleDateFormat.format(new Date()));
        }
    }

    public class DayHourPartitionRender extends PartitionRender {

        private SimpleDateFormat simpleDateFormat = new SimpleDateFormat("/'dt'=yyyy-MM-dd/'ht'=HH/HH-mm-");

        @Override
        public void render(StringBuilder base) {
            base.append(simpleDateFormat.format(new Date()));
        }
    }

}

```

通过一个简单的枚举解决。


