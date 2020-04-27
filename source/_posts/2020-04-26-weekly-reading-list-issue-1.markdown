---
layout: post
title: "Weekly Reading List Issue #1"
date: 2020-04-26 12:22:08 +0800
comments: true
categories: [wrl]
---

<!-- more -->

## FoundationDB Record Layer: A Multi-Tenant Structured Datastore

[[链接]](https://arxiv.org/abs/1901.04452)

FoundationDB 2015 年被 Apple [收购](https://techcrunch.com/2015/03/24/apple-acquires-durable-database-company-foundationdb)并于 2018 年[开源](https://www.foundationdb.org/blog/foundationdb-is-open-source)，作为 Apple 为数不多的开源项目受到广泛关注。简单介绍 FoundationDB 是一个基于 Paxos 的分布式 KV 存储，底层存储结构是 B-tree（是的，并不是 LSM tree），定位上跟 Google 的 Spanner 非常相似。这篇论文发表在 [SIGMOD 2019](https://sigmod2019.org/sigmod_industry_list)，介绍的是基于 FoundationDB 的 record-oriented 结构化存储框架（也已经[开源](https://github.com/FoundationDB/fdb-record-layer)）。[Layer](https://apple.github.io/foundationdb/layer-concept.html) 是 FoundationDB 一个很有特色的概念，在最基本的 KV 上无限扩展更加复杂的数据模型。这个框架整体上有几个亮点：

- 基于 Protocol Buffers 的数据模型定义
- 丰富的索引类型支持（单字段索引、嵌套字段索引、列表字段索引、聚合索引、rank 索引、全文索引、多字段联合索引等），并且索引是可以跨表的（这里简单将 record type 理解为表）。
- 基于 Java 的查询 API（并不是 SQL）

目前已经被应用在 [CloudKit](https://developer.apple.com/icloud/cloudkit)，替代旧的 Cassandra + Solr 架构（旧架构也有[一篇论文](https://dl.acm.org/doi/10.1145/3164135.3164138)介绍）。CloudKit 作为一个庞大的存储服务供所有 Apple 生态的应用和用户使用，这也就是论文标题中 Multi-Tenant 的含义。

## Introducing Dispatch

[[链接]](https://netflixtechblog.com/introducing-dispatch-da4b8a2a8072)

Incident 管理一直是 DevOps 领域比较热门的话题，Netflix 开源了他们自己的 incident 管理工具 [Dispatch](https://github.com/Netflix/dispatch)，更早之前 LinkedIn 也[开源](https://engineering.linkedin.com/blog/2017/06/open-sourcing-iris-and-oncall)过类似的东西。

## Agent57: Outperforming the human Atari benchmark

[[链接]](https://deepmind.com/blog/article/Agent57-Outperforming-the-human-Atari-benchmark)

大众对于 DeepMind 的认知恐怕就是[下下围棋](https://en.wikipedia.org/wiki/AlphaGo)、[打打星际](https://deepmind.com/blog/article/alphastar-mastering-real-time-strategy-game-starcraft-ii)，最近又搞起了雅达利的游戏，可以说是把强化学习玩儿出花儿了。最新一代的 Agent57 已经可以在全部 57 个游戏里战胜人类玩家。

## An Illustrated Guide to Graph Neural Networks

[[链接]](https://medium.com/dair-ai/an-illustrated-guide-to-graph-neural-networks-d5564a551783)

Graph Neural Networks（GNN）最近几年已经火得不行，Amazon 也开源了相关的框架 [DGL](https://www.dgl.ai/)。这篇文章以一种简单的示意图的形式介绍什么是 GNN，帮助不了解 GNN 的人建立一个简单的认知。

## Debugging with Delve

[[链接]](https://tpaschalis.github.io/delve-debugging)

[Delve](https://github.com/go-delve/delve) 是一个 Go 语言的 debugger，Go 官方也[推荐](https://golang.org/doc/gdb)优先考虑使用它而不是 GDB。这篇文章简单介绍了 Delve 的基本功能，其实跟 GDB 的使用方式很类似，但是 Delve 的亮点在于可以理解 Go 语言的语义以及调试 goroutine。

## gofiber/fiber

[[链接]](https://github.com/gofiber/fiber)

Fiber 是（又）一个 Go 语言的 HTTP 框架，设计上很大程度受了 Node.js 中非常流行的 [Express](https://expressjs.com) 启发（API 非常相似）。得益于底层使用的 [fasthttp](https://github.com/valyala/fasthttp) 库，在 Fiber 自己的[评测](https://docs.gofiber.io/benchmarks)中超越了很多市面上现有的框架。
