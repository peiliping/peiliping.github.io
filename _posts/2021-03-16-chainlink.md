---
layout: post
title:  "Chainlink"
date:   "2021-03-16 10:00:00"
categories: "chainlink"
permalink: /archivers/2021-03-16-chainlink
---


Chainlink目前已经是预言机领域里的顶流项目，无论是经济模型还是架构设计

都非常值得学习的。


## 预言机Oracle

因为以太坊区块链上无法发起对外部世界的请求，也就是无法直接获取外部数据，

极大的限制了智能合约的使用范围。将外部数据引入区块链世界，就是预言机的职能，

将拉取数据变为推送数据，保存在合约里。这一点与流计算中的恰好一次，其实是

非常相似的问题。如果你想让流计算可以反复执行，且结果是恰好一次的，那需要

严格控制堆外的依赖，比如Redis的计数器、外部系统的实时数据接口等。

解决外部依赖问题时，经常是将外部数据写入流中（或join）、在流引擎内保存状态。


## 成本问题

改为推送数据模式后，面临极大的成本问题。如果数据频繁发生变化，那就需要不断调用

合约的更新方法。但链上业务使用数据并不一定那么频繁。或者是某些数据使用不太频繁。

这个成本问题就极大的制约了预言机服务的范围，目前看几个主流预言机项目提供的数据

总共也不超过100种。而且更新频率都不是很高，一般10分钟左右。

除了写入成本以外，还有共识成本。比如早期的Chainlink所有节点都会把数据上报到合约中，

最后算一个中位数。每一轮都会有20个左右的节点参与。相当于把写入成本放大了20倍。

当然现在的Chainlink OCR已经把这个问题解决了，思路就是把共识环节放在低成本链去实现。


## 共识问题

预言机的数据真的可以依靠共识来解决么？这是个非常值得讨论的话题，虽然共识是区块链

类型的项目惯用的手法。但很多都是生搬硬套来的，并不是很恰当。API3项目的思路也不错，

预言机的数据由可信的权威机构来提供，绕过了关于数据正确的证明过程，只解决可用性问题。

Chainlink是依靠质押处罚、门限签名等手段来防止数据错误，再最新的计划里还将引入一些

博弈论的思路，比如囚徒困境等，增加被攻击的难度。显然这些手段都是有一定积极效果的，

但是为此付出的成本也是极大的，我个人并不看好这种发展思路。Chainlink未来的计划还是

很值得期待的，主要是在零知识证明方向上，可信计算和验证等技术。


## 服务模式

Chainlink采用的技术以去中心化为主，但是运用模式还是一个典型的中心化模式。

毕竟当前还是处于发展的早期阶段，希望未来的Chainlink可以越来越自由化。