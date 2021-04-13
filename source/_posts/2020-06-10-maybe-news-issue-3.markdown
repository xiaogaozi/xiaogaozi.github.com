---
layout: post
title: "Maybe News Issue #3"
date: 2020-06-10 17:37:27 +0800
comments: true
categories: [maybe news]
---

> 「Maybe News」是一个定期（或许不定期）分享一些可能是新闻的知识的[系列文章](/categories/maybe-news/)，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。你也可以通过[邮件订阅](https://maybe.news)它。

<!-- more -->

## Kudu: Storage for Fast Analytics on Fast Data

[[链接]](https://kudu.apache.org/kudu.pdf)

[OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing)（Online Analytical Processing）一直是大数据领域非常重要的应用场景，光有数据也不行，你得「分析」啊。自从有了 Hadoop，OLAP 的工具就一直在演变，从最早的裸写 MapReduce 任务，到 [Pig](https://pig.apache.org)、[Hive](https://hive.apache.org)、[Presto](https://prestosql.io)、[Impala](https://impala.apache.org)、[Druid](https://druid.apache.org)、[ClickHouse](https://clickhouse.tech)，以及今天要介绍的 [Kudu](https://kudu.apache.org)。一个明显的趋势是 OLAP 引擎在逐步朝着「去 Hadoop 化」和「实时化」发展，当然这些项目里最新的也已经是 2016 年发布的了，接下来会怎么变化还是个未知数。

先讲讲为什么会有类似 Kudu 这样的项目诞生。传统的 OLAP 引擎因为是构建在 HDFS 上的，要想分析数据首先得将数据存储到 HDFS 上，而这个过程（通常叫做 ETL）往往是比较耗时以及复杂的。同时由于 HDFS 天生不支持随机读写，为了弥补这个「缺陷」，有了 HBase 这样的项目。但 HBase 对于 OLAP 场景是不够友好的，因此往往需要把数据从 HBase 再导入到 HDFS 中，这个过程也可能比较耗时，维护成本也比较高。因此 Kudu 的目标是实现一个即支持随机读写（主要是写），又针对大批量查询进行优化的存储引擎。这种时候 HDFS 就显得很累赘了，这也是为什么越来越多引擎选择不依赖 HDFS 的缘故（Kudu 官网也在 FAQ 中专门[解释](https://kudu.apache.org/faq.html#why-doesnt-kudu-store-its-data-in-hdfs)了为什么不用 HDFS）。当然并不是说 HDFS 就没用了，有很多数据还是非常静态的，对于实时性要求也不高，此时用 HDFS 是一种简单经济的选择。

这篇论文虽然介绍的是 Kudu 早期的一些设计思想，但基本上属于最核心的功能。跟很多分布式数据库一样，Kudu 也是受 [Spanner](https://research.google/pubs/pub39966) 启发。系统架构上分为一个 Master 服务和若干 Tablet 服务。Master 负责维护元信息，包括 Tablet 节点和数据的。Tablet 服务则负责数据存储，每台节点上会有几十至数百个 tablet，每个 tablet 中包含了若干数据，最大可以达到几十 GB 的规模（这里你可以把 tablet 类比为很多别的系统中的 region 概念）。

跟很多关系数据库一样，Kudu 是有 table 的概念的。但跟很多 NoSQL 数据库不一样的地方是，强制用户必须显式定义 schema。Kudu 一个有意思的设计在于同时支持了 hash 和 range 这两种数据 partition 方法，而不像别的系统只支持其中一种（有关这两种 partition 的介绍可以看我之前的[一篇文章](https://blog.xiaogaozi.org/2020/05/25/how-to-design-a-distributed-index-framework-part-5/)）。这样设计的好处是即保留了 hash 的数据均匀分配特点，可以在一定程度上防止读写热点，又保留了 range 对于范围扫描的友好性。

Tablet 服务之间是通过 Raft 来进行数据复制，因此可以认为 Kudu 是一个保证强一致性的存储系统。值得注意的是 Kudu 的默认设置是 500 毫秒的心跳间隔以及 1.5 秒的选举超时，这个跟 Raft 论文推荐的时间相比长了不少（推荐的选举超时是 150~300 毫秒）。当集群扩容时，新节点将会首先进入 `PRE_VOTER` 状态，等到 log 追上以后再变成 `VOTER` 状态，这个设计也是 Raft 论文中建议的，不过论文中叫做 learner 或者 non-voting member。Master 服务虽然是单点设计（即状态不是分布式存储），但为了保障高可用也可以通过 Raft 实现多节点状态复制，只不过任意时间只能有一个节点工作。

Kudu 的数据存储引擎是完全自己设计的，没有直接用任何现有的引擎，虽然也能多少看出一些别的引擎的影子。关于这一点可以理解，OLAP 系统区别于 [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) （Online Transactional Processing）系统的最大不同即在于数据存储的形式，简单理解后者是行式（row-oriented）存储，而前者是列式（column-oriented）存储。著名的 [Parquet](https://parquet.apache.org) 就是广泛被用于 OLAP 场景的列式存储格式，Kudu 在实现上也复用了很多 Parquet 的代码。

每个 table 在存储级别会被分割为多个 RowSets，顾名思义每个 RowSets 是由很多行（row）组成，RowSets 之间不会有重复的数据，但主键的范围可能会交叉。

当有新的数据时会首先存储到内存中的 MemRowSets，底层实现是一个使用乐观锁（optimistic locking）的并发（concurrent）B 树。比较特别的一点是数据并不是一开始就按照列式进行存储，MemRowSets 中还是用的行式存储。当数据累积到一定程度 MemRowSets 就会持久化到磁盘上，称之为 DiskRowSets，每个 DiskRowSet 大小上限是 32MB。DiskRowSet 由两部分组成：基础数据（base data）和增量数据（delta stores）。

基础数据是列式格式，即每一列都单独连续存储，每一列内部又划分成了多个小的页（page），有一个 B 树根据行号索引了这些页。每一列可以由用户指定不同的编码（encoding）方法（如 dictionary encoding、bitshuffle、front coding），同时也可以使用通用的压缩算法对数据进行压缩（如 LZ4、gzip、bzip2），基于列的编码及数据压缩是列式存储非常大的一个特点。

增量数据也分为内存和磁盘两种形式。内存中的叫做 DeltaMemStores，这个跟 MemRowSets 的实现一样。磁盘中的叫做 DeltaFiles，是一个二进制类型的列块（column block）。不管是内存还是磁盘上的数据都会有一个额外的从 `(row_offset, timestamp)` 到 RowChangeList 的映射，`row_offset` 是某一行在一个 RowSet 中的偏移，RowChangList 是二进制编码以后的增量操作（如更新某一列、删除某一行）列表。同样的，DeltaMemStores 也会持久化到磁盘上变成 DeltaFiles。

这些增量数据会定期跟基础数据进行合并（compation），以防止过多的增量文件。同时 DiskRowSets 之间也会进行合并，目的是清理已经被删除的行以及减少 DiskRowSets 之间的主键交叉范围。

前面提到的将内存中的数据持久化到磁盘及对数据进行合并操作，都是由一组单独的后台任务来完成，但是什么时候执行什么操作是由一个调度器来控制的。有趣的是 Kudu 将调度器的逻辑抽象成了一个[背包问题](https://en.wikipedia.org/wiki/Knapsack_problem)，只不过需要权衡的不是背包容量，而是 I/O 带宽。

Kudu 本身只提供编程语言级别的 API（如 Java、C++），而 OLAP 系统中常用的 SQL 需要配合其它项目来实现。Kudu 原生已经跟 Spark 和 Impala 集成，也就是说你可以在这两个系统中通过 SQL 来查询。

最后是性能评测。在 [TPC-H](http://www.tpc.org/tpch) 数据集上与 Parquet 进行对比测试，Kudu 平均有 31% 的性能提升，尽管如此 Kudu 团队认为随着 Parquet 的迭代这个差距可能会逐渐缩小。在与 [Phoenix](http://phoenix.apache.org) 的对比测试中也有 16~187 倍的性能提升。最后与 HBase 进行的 [YCSB](https://github.com/brianfrankcooper/YCSB) 测试是为了看看 Kudu 在 OLTP 场景的性能，虽然它本身并不是为 OLTP 场景而设计，结果上的确也是 HBase 表现更好，但 Kudu P99 6 毫秒的响应时间在某些时候也许可以代替 OLTP 系统。

Kudu 由 Cloudera 公司开发并于 2016 年正式发布，现已捐献给 Apache 基金会，整个系统使用 C++ 语言编写。

顺带说个题外话这两年炒得比较火的 [HTAP](https://en.wikipedia.org/wiki/Hybrid_transactional/analytical_processing)（Hybrid Transactional/Analytical Processing），本质上是希望在一个引擎中同时适配 OLTP 和 OLAP 这两个场景。但在我看来就目前的技术现状这个愿景实现起来还是比较困难，软件工程界的名言[「没有银弹」](https://en.wikipedia.org/wiki/No_Silver_Bullet)告诉我们不存在一个可以通吃的、完美的方案，因此 HTAP 目前更多还只是一个营销概念吧。

## 字节跳动自研强一致在线 KV & 表格存储实践

[[上篇]](https://mp.weixin.qq.com/s/jdPE9WClBuimIHVxJnwwUw) [[下篇]](https://mp.weixin.qq.com/s/DvUBnWBqb0XGnicKUb-iqg)

[又](https://github.com/cockroachdb/cockroach)[又](https://github.com/pingcap/tidb)[又](https://github.com/dgraph-io/dgraph)[又](https://kudu.apache.org)[又](https://github.com/vesoft-inc/nebula)一个受 Spanner 启发的分布式存储（Google 功德无量！Jeff Dean 万寿无疆！），这次的项目来自字节跳动。关键词：range 分割、Raft、RocksDB、MVCC、分布式事务、SQL 层，看看这些也基本能对整体设计猜个八九不离十了，比较有价值的信息是学习学习字节跳动在他们的实践中的一些经验。项目使用 C++ 语言编写，目前没有开源。

## Challenges Supporting MIG in Kubernetes

[[链接]](https://docs.google.com/document/d/1Dxx5MwG_GiBeKOuMNwv4QbO8OqA7XFdzn7fzzI7AQDg)

随着深度学习的蓬勃发展，GPU 共享逐渐成为了 K8s 社区的一个[热门话题](https://github.com/kubernetes/kubernetes/issues/52757)。目前 NVIDIA 官方提供的[设备插件](https://github.com/NVIDIA/k8s-device-plugin)可以申请的最小资源粒度还是 1 个 GPU，但很多时候资源是浪费的。为了提升 GPU 的资源利用率，社区已经出现了多种解决方案，例如分别来自[阿里云](https://github.com/AliyunContainerService/gpushare-scheduler-extender)、[腾讯云](https://github.com/tkestack/gpu-manager)以及 [AWS](https://github.com/awslabs/aws-virtual-gpu-device-plugin) 的实现。现在 NVIDIA 官方终于在新一代的 Ampere 架构硬件上原生支持了共享，也就是标题中的 MIG（Multi-Instance GPUs）。这篇文档来自 NVIDIA 团队，首先介绍了当前是如何在 K8s 中管理 GPU 资源的，然后介绍了 MIG 的一些概念，最后提议了 4 个支持 MIG 的可能的解决方案。整体感觉 GPU 共享还是没有 CPU 灵活，不少地方设置了限制，但毕竟这是 NVIDIA 官方迈出的第一步。

## How to read deep learning papers?

[[链接]](https://www.reddit.com/r/MachineLearning/comments/gi3ihe/d_how_to_read_deep_learning_papers)

Reddit 上一个有趣的讨论：如何阅读深度学习的论文？我们常常调侃机器学习就是在「炼丹」，没有人能解释为什么结果就是有效的，反正[「it just works」](https://www.youtube.com/watch?v=YPN0qhSyWy8)。最高票的评论说你不需要接受论文中的每一个观点，只要把注意力集中在作者提供的证据并有选择性地调整你的想法就好了。正好前段时间前微软执行副总裁沈向洋博士做了一个主题名为[「You are how you read」](https://www.bilibili.com/video/BV1df4y1m74k)的演讲，主要内容就是一些阅读论文的经验（有趣的是沈博士在几年前还写过一篇叫做[「You are what you write」](https://www.linkedin.com/pulse/you-what-write-harry-shum)的博客）。

## Farewell, TensorFlow

[[链接]](https://mrry.github.io/2020/05/10/farewell-tensorflow.html)

TensorFlow 核心开发者、Google Brain 团队的 Derek Murray 宣布离开，这位大哥在 GitHub 和 Stack Overflow 上都很活跃，如果你经常浏览社区应该对他的头像不陌生。在这篇告别文中 Murray 提到了很多 Google 内部帮助工程师解决问题的工具，并详细介绍了近期对 TensorFlow 底层运行时进行的一项性能优化的过程，在部分评测中可以提升 10% 的推理性能。这个优化目前已经合入 master，并将在 TensorFlow 2.3 发布。
