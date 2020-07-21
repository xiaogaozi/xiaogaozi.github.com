---
layout: post
title: "Maybe News Issue #5"
date: 2020-07-21 14:08:28 +0800
comments: true
categories: [maybe news]
---

> 「Maybe News」是一个定期（或许不定期）分享一些可能是新闻的知识的[系列文章](/categories/maybe-news/)，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。你也可以通过[邮件订阅](https://digest.xiaogaozi.org/maybe-news)它。

<!-- more -->

## DynamicEmbedding: Extending TensorFlow for Colossal-Scale Applications

[[链接]](https://arxiv.org/abs/2004.08366)

在[第一期](https://blog.xiaogaozi.org/2020/05/21/maybe-news-issue-1/) Maybe News 中介绍了腾讯提出的解决 TensorFlow 中大规模稀疏特征模型训练的方案，本期的这篇论文来自 Google（准确说是 Google Smart Campaigns 团队）。作为发明 TensorFlow 的公司，Google 内部团队的设计思想值得借鉴。

这个系统被称之为 DynamicEmbedding（DE），名字简单直观，要解决的场景也是很多公司都遇到的如何动态维护 embedding。系统内部分为两个组件：DynamicEmbedding Master（DEM）和 DynamicEmbedding Worker（DEW），合起来叫做 DynamicEmbedding Service（DES）。DEM 负责处理所有客户端请求，包括 embedding 查找（lookup）、更新（update）等。DEW 负责 embedding 存储、梯度更新等，所有请求都来自 DEM。同时新增了几个 TensorFlow API，如 `dynamic_embedding_lookup()`、`compute_sampled_logits()`，这些 API 是整个系统的关键入口，任何模型在接入 DES 的时候都需要在特定的地方调用这些 API。以上设计看起来跟大部分公司的方案没有太大差别。

通过实现一个叫做 EmbeddingStore 的通用接口，DEW 后端支持对接多种类型的存储，例如 Protocol Buffers、GFS、Bigtable，比较巧妙地将大规模 embedding 存储时面临的扩展性和稳定性问题转移到了外部存储系统。当然因为多了一次网络请求是否会影响整体的训练效率这点有待商榷，论文中介绍 BigtableEmbedding 时提到会将数据同时存储到本地缓存和远端，猜测这里本地缓存的目的便是为了加速存储操作。

Embedding 更新这一步涉及到一些常用的梯度下降（gradient descent）算法，为了保持一致，DEW 内部实现了跟 TensorFlow 原生提供的优化器（optimizer）同样的逻辑，并且大部分代码是可以复用的。当训练数据时间跨度很大时（如数月或者数年），可能存在很多无效的特征或者一些需要特殊处理的特征。因此 DEW 在每次 embedding 更新的时候会同时统计这个 embedding 的更新频率，通过设定一个恰当的阈值来保证只有部分 embedding 会持久化到存储系统里，那些低频的数据便不会继续保存。除了统计频率这种方法，通过 bloom filter 也可以实现类似的效果。

Serving 的时候因为 embedding 都已经存储到了外部系统，所以 DEW 就没有必要存在了，只需要在本地部署 DEM 负责处理读请求。为了提升推理的性能，本地缓存肯定是少不了的，同时批量处理查询请求也是非常重要的。

实验评估阶段首先比较了和原生 TensorFlow 训练同样的模型、同样的超参是否会有指标上的差异，模型选择的是 Word2Vec，梯度下降算法选择的是 SGD、Adagrad 和 Momentum。从最终训练的 loss 上看几乎没有差别，说明 DE 系统不会对模型质量有影响。

接着测试了字典（dictionary）大小对模型精度（accuracy）的影响，理论上 DE 系统其实是不限定字典大小的，从实验的两个模型 Word2Vec 和 Sparse2Seq 上来看也的确是字典大小越大模型精度越高。

然后是评测模型训练时两个重要的系统指标：集群总的内存占用和每秒训练的 global steps（GSS）。分别测试了三个模型：Word2Vec，Image2Lable 和 Seq2Seq。在使用原生的 TensorFlow 时集群内存占用会随着 worker 数量的增大而显著增长（在 Word2Vec 模型中尤为明显），相比之下 DE 系统的内存占用只跟 embedding 的总大小有关，与 DEW 的数量无关。之所以有这样的差异也是因为原生的 TensorFlow 会在不同 worker 间重复存储 embedding 数据。GSS 的对比上两者的加速比都差不多，但是总体上 DE 还是会更优。

最后论文中详细介绍了 DE 在 Google Smart Campaigns 产品中的一个重要应用：给广告主自动推荐投放的关键词。这是一个叫做 Sparse2Label 的模型，输出即是推荐的关键词（label）。这个模型带来的最大变化是以前需要针对每一种语言训练一个单独的模型，而现在只需要一个模型即可。通过对比一些核心指标（如 CTR），DE 推荐的关键词都明显更好。整个模型也是随着时间不断增长的，截止 2020 年 2 月这个模型的参数量已经达到了 1249 亿个，如果每个参数按 4 字节算的话模型大小差不多为 465 GiB（其实比想象中小）。

另一个更难评估的指标是用户搜索的关键词（query）与广告投放的关键词之间的关联度，很多时候两者之间并不是完全匹配的。作者是通过人工评估 38 万个样本的方式来解决的，每个样本都会有 5 个人类进行打分，分数区间从 -100 到 100，越高越匹配，然后计算这 5 个分数的平均值作为这个样本的最终分数。大于等于 50 分的样本认为是好（good）的样本，小于等于 0 分的则认为是不好（bad）的样本，前者除以后者被称作 GB ratio，这个比率越大越好。每个推荐的关键词都会同时有一个置信值（也就是网页和关键词 embedding 之间的 cosine 距离），从评测结果上来看当这个置信值大于 0.7 时，不好的样本量将会显著减少。实际生产环境收集的数据也印证了 DE 系统推荐的关键词是 GB ratio 最高的。

总结一下 DE 系统解决了原生 TensorFlow 在大规模 embedding 模型训练时效率低下（甚至不可用）的问题，短期内这个系统估计也不会开源或者合并到上游。目前可以期待的还是腾讯的方案，他们已经提交了[代码](https://github.com/tensorflow/tensorflow/pull/41371)到 TensorFlow 社区。

## The Next Step for Generics

[[链接]](https://blog.golang.org/generics-next-step)

在[第二期](https://blog.xiaogaozi.org/2020/06/02/maybe-news-issue-2/) Maybe News 中曾经介绍过 Go 语言开发者关于泛型设计的一些思考，近期 Ian Lance Taylor 又和社区同步了一下最新进展。最大的变化就是去掉了 contract 这个新增的概念，改为复用 interface。同时创建了一个新的 [playground](https://go2goplay.golang.org)，可以方便大家试验泛型代码。如果最新的这一版设计社区没有太大异议的话，乐观估计将会在 Go 1.17 加入泛型特性，也就是在 2021 年 8 月左右。当然最终实现这个目标还是有很多的不确定性，特别是当前疫情对于全球影响的情况下。

## Fiber: Distributed Computing for AI Made Simple

[[链接]](https://eng.uber.com/fiberdistributed)

分布式计算在 AI 领域的需求一直都很强烈，但分布式计算不仅仅是将单机迁移到多机这样就足够了，还需要考虑如：易用性（降低用户从单机迁移的成本）、稳定性（自动容错）、弹性伸缩（和底层资源调度层配合）、线性加速（横向扩展多少机器就能带来多少性能提升）。Uber 和 OpenAI 共同开发的 Fiber 框架便是尝试解决以上问题的一个例子，从 Fiber 的[论文](https://arxiv.org/abs/2003.11164)能看到这个框架最初设计面向的是强化学习（Reinforcement Learning）场景，在这个领域有很多类似的框架，比如 Google 的 [Dopamine](https://github.com/google/dopamine)、Facebook 的 [ReAgent](https://github.com/facebookresearch/ReAgent)、UC Berkeley 的 [Ray](https://github.com/ray-project/ray)。自动容错和弹性伸缩这两个特性又让我联想到蚂蚁金服的 [ElasticDL](https://github.com/sql-machine-learning/elasticdl) 和才云的 [FTLib](https://github.com/caicloud/ftlib)。

## The impact of slow NFS on data systems

[[链接]](https://engineering.linkedin.com/blog/2020/the-impact-of-slow-nfs-on-data-systems)

LinkedIn 分享了他们使用 NFS 进行数据库备份时遇到的性能问题，因为备份进程和数据库进程是一起部署的，因此这个问题还间接影响到了在线业务的稳定性。整个问题分析过程清晰易懂，还能顺便复习一下大学里学习的计算机网络和操作系统的一些知识。但问题的根源 NFS 服务的性能为什么这么差还是没有特别好的解决方案，可能在这个场景里 NFS 就不是特别好的选择吧。

## Kubeflow & Kale simplify building better ML Pipelines with automatic hyperparameter tuning

[[链接]](https://medium.com/kubeflow/kubeflow-kale-simplify-building-better-ml-pipelines-with-automatic-hyperparameter-tuning-5821747f4fcb)

Jupyter Notebooks 是当下数据科学家或者算法工程师日常工作非常重要的一个组件，交互式的界面加上即时的代码运行反馈极大地提升了开发效率。但是如果要将机器学习任务提交到集群中运行往往还得依靠类似 Kubeflow Pipelines 这种 DAG 管理及调度组件，Kubeflow Pipelines 有一套基于 Python API 的语法，因此用户需要再重新定义一个独立的 pipeline。有没有办法直接将 notebook 中已经验证过的代码自动转换成 pipeline 并提交到集群呢？[Kale](https://kubeflow-kale.github.io)（**K**ubeflow **A**utomated Pipe**L**ines **E**ngine）即是为了解决这个问题而诞生，它是一个能够将 Jupyter Notebooks 自动转换为 Kubeflow Pipelines 的工具。在最新的 0.5 版本中 Kale 新增了对 [Katib](https://github.com/kubeflow/katib) 的集成，后者是进行自动超参调优（Hyperparameter Tuning）和神经网络架构搜索（Neural Architecture Search）的组件。

## GoogleCloudPlatform/spark-on-k8s-operator #976: Add support for dynamic allocation via shuffle tracking

[[链接]](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/pull/976)

Spark 3.0 新增了动态伸缩 executor 的特性（默认关闭），具体实现可以查看 [SPARK-27963](https://issues.apache.org/jira/browse/SPARK-27963)，用户使用文档可以查看[这里](http://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation)。用户需要预先设定诸如初始、最小和最大 executor 数量这样的参数，之后 Spark 运行时会根据当前任务排队时间和 executor 空闲时间这些指标去创建或者销毁 executor。对于有状态的 executor（如 shuffle 时存储到磁盘的数据、cache 到内存和磁盘的数据）会有一些特殊的策略防止错误回收资源，如设置一个 executor 持有 shuffle 数据的超时时间，或者直接使用外部 shuffle 服务。spark-on-k8s-operator 项目近期也支持了这个特性，可以直接通过 YAML 配置来开启，不过遗憾的是动态资源分配目前还不支持 K8s 模式。

## Boiled Hippo

[[Bandcamp]](https://spacefruityrecords.bandcamp.com/album/boiled-hippo-2) [[网易云音乐]](https://music.163.com/#/album?id=91278378) [[虾米]](https://www.xiami.com/album/1ttwrEdcce1)

本期最后推荐一张来自我的一个好朋友的唱片，Boiled Hippo 是一支北京的迷幻摇滚乐队，经过多年的演出积累终于在今年发行了乐队的第一张同名专辑。虽说是迷幻摇滚，但如果从旋律上讲绝对是非常「好听」的。如果你有兴趣购买实体唱片（黑胶、磁带、CD 都有），目前可以在北京的 fRUITYSPACE、fRUITYSHOP、独音唱片，上海的 Daily Vinyl，金华的 Wave 这几个地方购买。
