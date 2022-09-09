---
layout: post
title:  "write file"
date:   "2021-07-28 10:00:00"
---


最近在优化一些shell脚本任务，涉及到对文件的读写过程，这里聊聊我碰到的一些问题。

我的脚本主要是对数据进行统计和计算，数据具有时序性，每分钟会有一行结果存在结果文件中。

脚本每分钟执行一次，每次计算上一分钟和当前分钟的结果。保存结果数据时，会涉及到删除结果文件的最后一行，因为最后一分钟会被修正。


## 如何衔接两次任务的时间

数据结果的文件每一行都对应一分钟的数据，每行开头都记录对应的时间戳。

每次任务开始时，都读取结果文件的末尾，计算合理的数据时间。

```

thishourday=`date "+%Y-%m-%d"`
startTime=`date -d "${thishourday}" +%s`

if [ -f "$rfile" ]; then
  startTime=`tail -n 1 $rfile | awk '{print $1}'`
else
  touch $rfile
fi

```


## 原始数据文件有DailyRolling

这就意味着每天00:00处理数据时，需要读取前一天的文件和当天文件。

```

starthourday=`date --date=@${startTime} "+%Y-%m-%d"`

if [ $starthourday = $thishourday  ];then
  dfile=$dfile
else
  dfile=$dfile"-"$starthourday" "$dfile
fi

```


## 保存计算结果

每次计算任务完成后，都先将数据写入临时文件tfile中，接下来我们删除结果文件的最后一行。

删除我们用了一个简单的命令

```

sed -i '$ d' $rfile

```

最后我们将临时文件Merge到结果文件的末尾，删除临时文件。

```

cat $tfile >> $rfile
rm -rf $tfile

```

## 性能问题

如果结果文件越来越大，这个任务会越来越慢，主要的性能问题在删除文件最后一行这一步。

sed删除最后一行这样的方式，几乎等于把整个文件重写了一遍。有兴趣的朋友可以看看这篇[Blog](https://www.codenong.com/4881930/)。

我觉得出现这个问题本身就代表了设计问题，临时数据（最后一行）是否要直接写入结果文件？

可以借鉴Compaction那样的思路，将不确定的临时数据写入临时文件，当可以确定时写入正式的结果文件中,规避性能问题。

最好还是把这些数据存入数据库吧，时序数据库千千万。