---
layout: post
title: "Maybe News Issue #1"
date: 2020-05-21 17:34:22 +0800
comments: true
categories: [maybe news]
---

> 「Maybe News」是一个定期（或许不定期）分享一些可能是新闻的知识的[系列文章](/categories/maybe-news/)，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。你也可以通过[邮件订阅](https://digest.xiaogaozi.org/maybe-news)它。

<!-- more -->

## CFS: A Distributed File System for Large Scale Container Platforms

[[链接]](https://dl.acm.org/doi/10.1145/3299869.3314046)

跟[上次介绍](https://blog.xiaogaozi.org/2020/04/26/weekly-reading-list-issue-1/)的 FoundationDB Record Layer 一样，这篇来自京东团队的论文也是发表在 SIGMOD 2019，介绍了一个为大规模容器平台设计的分布式文件系统。

系统整体由 3 部分组成：元数据子系统（metadata subsystem）、数据子系统（data subsystem）、资源管理器（resource manager）。元数据子系统负责维护 inode 和 dentry（directory entry），数据子系统负责存储数据块，资源管理器负责处理客户端的各种文件操作请求以及维护前面两个子系统的状态。元数据子系统和数据子系统都是多 partition 的分布式系统，多个元数据和数据的 partition 逻辑上共同组成一个卷（volume），这个卷即是对客户端（容器）可见的存储单元并且可以被挂载，通过传统的 POSIX 接口访问。

因为上述 3 部分组件内部其实都是一个分布式系统，因此都用到了 Raft 作为一致性协议，资源管理器还用到了 RocksDB 作为本地持久化存储。稍微特殊的是数据子系统根据不同类型的写操作选择了不同的复制方案，论文里把这个叫做 Scenario-Aware Replication，具体讲就是顺序写操作（比如 append）用的是 primary-backup，而覆盖（overwrite）操作用的是 Raft。

系统的另一个亮点是基于资源利用率的 partition 分配策略，论文中叫做 Utilization-Based Placement。传统的 partition 分配策略通常是哈希，这种策略的优点是简单但是当扩缩容时必须进行 rebalance。CFS 的做法是元数据和数据子系统定期上报内存、磁盘使用率到资源管理器，当需要创建新的 partition 时根据资源利用率选择最低的那个节点，这样设计的好处是不再需要 rebalance。但是对于这种设计方案是否会造成数据不均衡表示存疑，论文中也没有做过多论述。

为了尽量减少客户端的网络交互，不让某个系统组件成为瓶颈，客户端会缓存元数据子系统、数据子系统和资源管理器的信息到本地，当执行文件操作时会优先读取本地缓存。当然某些组件（比如资源管理器）还是有可能在某一天成为瓶颈，但是基于京东的经验这件事情基本上不会发生。

在与 Ceph 的评测中，CFS 平均有 3 倍的 IOPS 提升，特别是多客户端和随机读写的场景。这很大程度上得益于元数据和数据节点分离的设计，且 CFS 的元数据是全内存存储，而 Ceph 并不是。

分布式文件系统一直都是比较重要的基础组件，在分布式数据库、大数据、机器学习领域有广泛应用。常见的分布式文件系统如 HDFS、Ceph，在如今这个全面推行容器化的时代越来越显得捉襟见肘。容器化一个很大的特点是快速扩缩容，传统的存储系统在这一点上是非常不友好的，因此才会有越来越多针对容器化场景的基础组件诞生（具体可以访问 [CNCF](https://www.cncf.io) 查看），这里介绍的 CFS 是一个例子，另一个类似的是 [JuiceFS](https://juicefs.com)。

CFS 目前属于 CNCF 下的 [sandbox 项目](https://www.cncf.io/sandbox-projects)，且已经[开源](https://github.com/chubaofs/chubaofs)，使用 Go 语言编写。

## tensorflow/community #237: RFC: Sparse Domain Isolation for Supporting large-scale Sparse Weights Training

[[链接]](https://github.com/tensorflow/community/pull/237)

推荐系统大规模稀疏特征分布式训练一直是工业界一件有挑战的事情，大公司内部自研的训练框架大多已经解决了这个问题，但是在开源社区问题仍然存在。TensorFlow 作为也许目前最流行的深度学习训练框架，社区里也早有相关的讨论（比如 [#19324](https://github.com/tensorflow/tensorflow/issues/19324)、[#24539](https://github.com/tensorflow/tensorflow/issues/24539)、[#24915](https://github.com/tensorflow/tensorflow/pull/24915)），但基本都以烂尾告终。最新的 RFC #237 来自腾讯，区别于现有的一些开源实现（比如阿里巴巴的 [XDL](https://github.com/alibaba/x-deeplearning)、字节跳动的 [BytePS](https://github.com/bytedance/byteps)、蚂蚁金服的 [ElasticDL](https://github.com/sql-machine-learning/elasticdl)）完全自己重新造了一个 parameter server，腾讯的方案最大限度复用了 TensorFlow 现有的组件，对用户的代码侵入也最小。目前这个 RFC 还在讨论中，有兴趣可以订阅 PR。

## 深入云原生 AI：基于 Alluxio 数据缓存的大规模深度学习训练性能优化

[[链接]](https://mp.weixin.qq.com/s/2Pj8erPbYuMo7mBJvweJgQ)

机器学习模型训练由于依赖大量的数据作为输入，因此数据 I/O 的性能会直接影响模型训练的效率。有时间会发现计算设备的算力升级了，但是数据 I/O 跟不上了，反而拖慢了整个训练流程。阿里云团队分享的这篇文章便是他们在使用 Alluxio（试图）加速数据 I/O 的过程中的经验，虽然最后的优化结果性能指标其实也只是基本跟本地读取持平。

## Rob Pike interview: “Go has indeed become the language of cloud infrastructure”

[[链接]](https://evrone.com/rob-pike-interview)

没啥好介绍的了，值得一读的一篇采访。文中有两个有趣的问题：

- **对于 Rust 宣称的「没有 GC」的设计有什么看法**：Rob Pike 只是表示了他对 Rust 很感兴趣，其它意见不便发表。
- **如果可以时间旅行到最初设计 Go 的时候想给自己一个什么忠告**：无视那些仇恨者（haters），只需要聆听那些理解以及和你有共同目标的人的声音。不可能每一个人都认同你正在做的事情，但是那些鼓励你前进的人会是提供给你非常棒（fantastic）的想法、能量和灵感的源泉。

## 孤芳「自赏」：盯鞋音乐的前世与今生

[[上]](https://www.gcores.com/articles/121368) [[下]](https://www.gcores.com/articles/123770)

这两篇文章来自「竟然还能聊游戏」的机核，相对系统地介绍了「盯鞋（shoegaze）」这种音乐风格，作为目前可能是除了后朋克以外我最喜欢的音乐风格非常高兴能够有人科普，稍微欠缺的是文中没有提到任何中国的乐队。
