---
layout: post
title:  "websocket"
date:   "2020-06-29 10:00:00"
categories: "websocket netty"
permalink: /archivers/2020-06-29-websocket
---


最近一个多月在和websocket打交道，这次来总结一下最近一阶段的工作吧。


## hydra

三月份写了一篇关于netty的blog，介绍了我写的一个小项目叫[hydra](https://github.com/peiliping/hydra)。

经过这几个月的完善，已经可以作为一个websocket的压测工具了。

这两次压测实践中，单实例可以施压5W个并发连接数，满足大多数测试需求。

通过参数可以配置心跳消息、订阅指令等，满足业务测试的需求。


## watchdog

使用netty作为websocket的client，如果解决重连重试的问题呢？

在实际业务中，要考虑网络不稳定导致的websocket中断问题。websocket中断后，

要重新建立连接，还要将业务的订阅指令重新发送一遍。

在netty的中断分为两种情况，一个是创建连接就直接失败了，一种是在运行中突然中断。

这两种情况分别有回调的接口可以捕获，而后触发延迟重连。可以参考[watchdog](https://github.com/peiliping/watchdog)的实现。


## Bark

最近无意间发现了一个iOS的App叫Bark，可以自定义手机通知提醒的消息。

举个例子，你写了一个爬虫监控实时获取某只股票的价格，当他发生剧烈波动的时候，

给自己的手机发送一条提醒，来提醒自己关注股票。那么如何给手机发送免费的消息呢？

Bark就可以满足这个需求，安装Bark的App后，你会获得一个Url（包含一个uuid）。

执行这个Url，你的手机就会收到来自Bark的通知消息了，消息的内容来自Url参数。


## 提醒助手

将watchdog和Bark结合在一起，实现了一个小功能。

订阅Huobi的BTC行情信息，经过计算，将满足剧烈波动条件的消息通过Bark发送到手机上。