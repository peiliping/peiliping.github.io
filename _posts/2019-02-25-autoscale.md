---
layout: post
title:  "自动伸缩"
date:   "2019-02-25 10:00:00"
categories: "scale auto fregata"
permalink: /archivers/2019-02-25-scale
---

这次讲一下Fregata重构中的一个重要内容-自动伸缩，主要是解决两个方向的问题：

1、突发流量需要人工介入，不及时也太耗费人力

2、周期性波动的数据处理，在波峰波谷时不同处理方式

为了解决这些问题，我们在重构的Topology基础上，增加了伸缩容功能，可以增减

Parser和Sink的个数。下面介绍一下伸缩功能的一个基础，那就是如何判断伸缩。


## 状态

### 基本状态

在讨论这个功能时，我们一度非常困惑于如何对状态进行定义和划分还有转化。

我们阅读了很多关于状态机的文章和demo，定了4个基础状态：

1、固定状态（不可以缩、也不可以扩）

2、最大（不可以扩，可缩）

3、最小（不可以缩，可扩）

4、中间状态（可扩，可缩）

Topology中的Parser和Sink都有自己各自的状态，独立计算互不影响。

按照其并行度的最小值、最大值、当前值来作为定义状态的基础参数


```
	this.statesRules.add(new FixedStatus(super.componentType, 1, 1, 1));
        this.statesRules.add(new MinStatus(super.componentType, 3, 1, 1));
        this.statesRules.add(new BriskStatus(super.componentType, 3, 1, 2));
        this.statesRules.add(new MaxStatus(super.componentType, 3, 1, 3));
```

### 匹配、转化

我们的状态匹配方式就是最简单的遍历

```
        for (IStatus status : this.statesRules) {
            IStatus result = status.matchStatus(upperBound, lowerBound, cur);
            if (result != null) {
                return result;
            }
        }
```

### 趋势

每个周期的监控指标都会输入到当前状态中，指标的计算会得出一个当前的趋势（伸、缩、不动）。

在当前状态中，保持一个时间序列的趋势集合，当连续N次趋势产生时，就会触发状态的改变，也会

触发一个Action事件。每种状态因为其特点不同，对趋势的处理也会有所不同。

## 伸缩分类

伸缩我们主要分为两类：

1、Topology的伸缩，也就是增减并行度。

2、K8s的Deployment的伸缩，也就是增减副本数。

### Topology内

我们是优先对Topology进行调整，这样的代价是最小的。

### Docker副本数

当Topology已经是最小时，就考虑适当的减少Docker副本，进一步释放资源。

## 收益

无论是哪种伸缩，都可以帮助我们在流量增大时，自动提升处理能力去应对，在数据量小的时候，减少对

外部的负担，减少Tcp连接数、HDFS小文件数、提高数据的密度等。
