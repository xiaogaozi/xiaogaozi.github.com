---
layout: post
title: "Maybe News Issue #2"
date: 2020-06-02 09:25:45 +0800
comments: true
categories: [maybe news]
---

> 「Maybe News」是一个定期（或许不定期）分享一些可能是新闻的知识的[系列文章](/categories/maybe-news/)，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。你也可以通过[邮件订阅](https://digest.xiaogaozi.org/maybe-news)它。

<!-- more -->

## In Search of an Understandable Consensus Algorithm (Extended Version)

[[链接]](https://raft.github.io/raft.pdf)

终于有机会仔细阅读一遍 Raft 的论文，如果你还不了解 Raft 是什么可以看看我过去的一篇介绍分布式系统基础概念的[文章](https://blog.xiaogaozi.org/2020/05/25/how-to-design-a-distributed-index-framework-part-5/)。

Raft 为节点定义了三种状态：leader、follower 和 candidate（以及一个非正式状态 learner 或者叫做 non-voting member）。一个集群只会有 1 个 leader，其余节点都是 follower。Leader 负责处理所有的读写请求，如果请求 follower 会失败并告知客户端 leader 的地址。

每个节点都有一个自己的 log，log 中每个条目都有一个下标（index）。这个 log 基本算是 append-only 的，通常也需要持久化到可靠的存储上（例如磁盘）。当处理写请求时 leader 会首先更新自己的 log，然后通过 RPC 复制到其它节点，只要大多数（majority）节点更新成功 leader 就会认为这个请求已经 committed，此时会更新自己的状态机（state machine）并返回给客户端。如果 RPC 请求失败 leader 会不断重试直到成功。

如果出现异常，如 leader 宕机、网络故障等，就可能触发 leader 重新选举。选举过程是所有 follower 为 candidate 投票，只要获得多数票 candidate 就会升级为 leader。如果投票失败会继续新一轮选举，选举过程通常是毫秒级的。每一轮新的选举都会产生一个对应的 term（任期），Raft 在协议上保证了重新选举后的新 leader 一定是包含之前所有 term 已经 committed 的 log，这样就避免了新 leader 选举成功以后需要首先补上缺失的数据。

当集群需要伸缩时，leader 会首先将旧集群配置（configuration）和新集群配置合并到一起并通过 log 的形式复制到 follower。成功收到这个合并后配置的节点会用这个配置替代老的配置。一旦这个合并后的配置 committed，leader 就会创建一个只包含新配置的 log 继续复制到 follower。等到新的配置 committed，旧配置将不再生效，需要下线的节点也可以被安全关闭。

随着时间增长，log 的容量会越来越大，Raft 引入了快照（snapshot）机制，定期将 log 压缩到快照文件。这个快照文件同时也可以帮助新加入的节点快速补上缺失的数据。

总结一下 Raft 算法保证了以下几个属性始终成立：

- **Election Safety**：在一个特定的任期最多只能有一个 leader 被选举出来
- **Leader Append-Only**：leader 永远不会覆盖或者删除 log 中的条目，只会追加新的条目。
- **Log Matching**：如果两份 log 同时包含一个具有相同任期数和下标的条目，那么这两份 log 中这个下标之前的所有条目都应该是一致的。
- **Leader Completeness**：如果某个任期中的一个 log 条目已经 committed，那么在之后任期中选举出的新 leader 一定包含这个条目。
- **State Machine Safety**：如果一个节点已经将一个给定下标的 log 条目更新到自己的状态机，那么其它节点上同样的下标一定不会是不同的条目，也就是说不会更新一个不同的条目到自己的状态机。

更多有关 Raft 的信息可以查看它的[官网](https://raft.github.io)，强烈建议初次接触一致性协议的朋友看看网站上的动画演示，非常有助于建立一个形象直观的认知。

## Scaling Raft

[[链接]](https://www.cockroachlabs.com/blog/scaling-raft)

作为前面介绍 Raft 的一篇衍生阅读，原始的 Raft 实现是将所有节点看作一个 group，这种设计在某些场景（例如集群规模很小）是可行的。但是当集群规模大到一定程度，或者类似 [CockroachDB](https://github.com/cockroachdb/cockroach) 和 [TiKV](https://github.com/tikv/tikv) 这种将数据划分为非常多的 range，多个 range 组成一个 Raft group 的场景（通常叫做 Multi-Raft），就会发现 Raft 的基础网络通信已经足以影响单节点的性能（比如过多的心跳请求）。因此社区已经针对这样的问题有了一些优化方案，比如 [CockroachDB 的方案](https://github.com/cockroachdb/cockroach/issues/357)和 [TiKV 的方案](https://github.com/tikv/tikv/pull/4591)。这两个方案都很类似，基本思想是暂停那些不活跃的 Raft group 的网络通信，等到需要的时候再唤醒。

## Why Generics?

[[链接]](https://blog.golang.org/why-generics)

这篇文章是 Ian Lance Taylor 在 GopherCon 2019 演讲的文字版（文章中也附带了视频），主要介绍了目前 Go 的核心开发者关于泛型（generics）的一些思考。总的来说 Go 核心团队的设计思想还是保持 Go 语言一贯的简洁，不希望引入过多的概念和复杂性。大部分新增的语法特性都由提供泛型接口的开发者来学习，对于使用者来说和调用普通接口几乎没有区别。早在 2016 年社区就已经有了 [#15292](https://github.com/golang/go/issues/15292) 这个关于泛型的讨论，并且还在持续更新中，目前已经有了 710 条评论，Ian Lance Taylor 也在其中积极回复。虽然这个 issue 打上了 Go2 的标签，但泛型特性是否能在 Go 语言的 2.0 版本中出现现在还是个未知数。

## The Open Application Model from Alibaba’s Perspective

[[链接]](https://www.infoq.com/articles/oam-alibaba)

阿里云和微软在去年[共同宣布](https://cloudblogs.microsoft.com/opensource/2019/10/16/announcing-open-application-model)了 Open Application Model（OAM），OAM 组织的[核心成员](https://github.com/orgs/oam-dev/people)同时也是前 CoreOS 团队成员以及 etcd、K8s Operator 的创造者。简单理解 OAM 就是希望将传统的 K8s YAML 配置抽象成两部分：开发者和运维，开发者的配置中只包含与业务最相关的内容，而运维的配置中则包含与运行环境相关的内容。本质上是希望将开发者和运维的界线分得更清楚，让不同的角色更专注于自己的领域。在我看来 OAM 的好处当然是降低了普通开发者接入 K8s 的门槛，所谓大道至简，但这种表面上的「简」背后隐藏的复杂性也是不能忽略的。理想情况是某个云服务商能够完全包办所有跟运维有关的事情，用户只需要负责业务开发就好了。但现状还是不管多小的公司都肯定会有专人在负责运维工作。很多年前 Google App Engine 刚诞生时让所有人都眼前一亮，都认为这才是软件开发的未来啊，但即使是 Google 也没能让这个趋势持续下去。最近几年这个趋势又开始回潮，只不过换了一个名字叫做「Serverless」，希望这一次能够持续下去，虽然还有很长的路要走。

## Lightweight coscheduling based on back-to-back queue sorting

[[链接]](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/kep/20200116-lightweight-coscheduling-based-on-back-to-back-queue-sorting.md)

自从 K8s 1.15 新增了 [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework) 以后，原生调度器的扩展性有了很大程度的增强。这个 KEP 来自阿里云团队，提出了基于 Scheduling Framework 来实现 coscheduling（或者叫做 gang scheduling）。Coscheduling 这个特性对于机器学习任务来说是非常重要的，一个任务通常包含多个 pod，只有当多个 pod 能够同时运行时这个任务才算是正常运行，如果只有部分 pod 可以运行其实是一种资源的浪费。因此 coscheduling 保证的就是一个任务必须满足一定数量的 pod 都能够被调度时才会实际分配资源。这个特性在 K8s 社区早有讨论，也诞生了一些相关联的项目，如 [Volcano](https://volcano.sh)（前身是 [kube-batch](https://github.com/kubernetes-sigs/kube-batch)）。5 月初这个插件的第一版已经被 [merge](https://github.com/kubernetes-sigs/scheduler-plugins/pull/4) 到 scheduler-plugins 项目。

## Scheduler Support for Elastic Quota Management

[[链接]](https://docs.google.com/document/d/1ViujTXLP1XX3WKYUTk6u5LTdJ1sX-tVIw9_t9_mLpIc/edit?usp=sharing)

同样是与 K8s 相关的一个讨论，也同样来自阿里云团队。`ResourceQuota` 是 K8s 目前提供的一种限制某个 namespace 最大资源使用量的方式，但是在实际的多租户场景中，`ResourceQuota` 往往显得不够灵活。很多时候我们是希望给每个租户一个可以保证（guarantee）的最小资源量，以及一个超卖的最大资源量。当某个租户的资源比较空闲时，就允许其它租户临时租用。但是调度器也要保障这个租户有能力在必要的时候可以拿回这些被租用的资源，这通常是通过抢占（preemption）的方式来实现。这个提案就提出了扩展 `ResourceQuota` 来实现类似功能的想法。
