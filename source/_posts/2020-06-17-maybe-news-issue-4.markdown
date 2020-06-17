---
layout: post
title: "Maybe News Issue #4"
date: 2020-06-17 14:07:52 +0800
comments: true
categories: [maybe news]
---

> 「Maybe News」是一个定期（或许不定期）分享一些可能是新闻的知识的[系列文章](/categories/maybe-news/)，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。你也可以通过[邮件订阅](https://digest.xiaogaozi.org/maybe-news)它。

<!-- more -->

## AliGraph: A Comprehensive Graph Neural Network Platform

[[链接]](https://dl.acm.org/doi/10.1145/3292500.3340404)

AliGraph 是阿里巴巴团队研发的 GNN（Graph Neural Network）分布式训练框架（虽然标题里是「平台」但感觉还算不上），论文发表在 KDD 2019 和 PVLDB 2019。

论文开篇便提出了当下 GNN 模型训练的 4 个挑战：

1. 如何提高大规模图模型的训练效率及优化空间占用？
2. 怎样优雅地将异构（heterogeneous）信息组合到一个统一的 embedding 结果中？
3. 如何将结构化的拓扑（topological）信息与非结构化的属性（attribute）信息统一来共同定义那些需要保留的信息？
4. 如何设计一个高效的增量更新动态图的 GNN 方法？

后面的篇章便是详细介绍 AliGraph 如何解决以上这 4 个问题。框架从上至下整体分为 3 层：算子（operator）、采样（sampling）、存储（storage）。算子层包含常见的 GNN 运算操作，采样层包含几种预设的采样算法，存储层主要关注如何高效对大规模图进行分布式存储。在这 3 层基础之上可以实现任意的 GNN 算法以及应用。

存储层因为是要解决一个图的分布式存储问题，因此首先要将图进行分割（partition）。AliGraph 内置了 4 种图分割算法：[METIS](https://dm.kaist.ac.kr/kse625/resources/metis.pdf)、[顶点切割和边切割](https://www.usenix.org/conference/osdi12/technical-sessions/presentation/gonzalez)、[2D 分割](https://dl.acm.org/doi/10.1145/2503210.2503293)、[流式分割](https://dl.acm.org/doi/10.1145/2339530.2339722)。这 4 种算法分别适用于不同的场景，METIS 适合处理稀疏（sparse）的图，顶点切割和边切割适合密集（dense）的图，2D 分割适合 worker 数量固定的场景，流式分割通常应用在边（edge）频繁更新的图。用户需要根据自己的需求选择恰当的分割算法，当然也可以通过插件的形式自己实现。

另一个存储层关心的问题是如何将图结构和属性（attribute）共同存储。这里讲的图结构即顶点和边的信息，这是最主要的图数据。同时每个顶点也会附加一些独特的属性，例如某个顶点表示一个用户，那附加在这个用户上面的属性就是类似性别、年龄、地理位置这样的信息。如果直接将属性信息和图结构一起存储会造成非常大的空间浪费，因为从全局角度看同一种类型的顶点的属性是高度重合的。并且属性与图结构的大小差异也非常明显，一个顶点 ID 通常占用 8 字节，但是属性信息的大小从 0.1KB 到 1KB 都有可能 。因此 AliGraph 选择将属性信息单独存储，通过两个单独的索引分别存储顶点和边的属性，而图结构中只存储属性索引的 ID。这样设计的好处自然是显著降低了存储所需的空间，但代价就是降低了查询性能，因为需要频繁访问索引来获取属性信息。AliGraph 选择增加一层 LRU 缓存的方式对查询性能进行优化。

存储层关心的最后一个问题也是跟查询性能有关。在图算法中一个顶点的邻居（neighbor）是非常重要的信息，邻居可以是直接（1 跳）的也可以是间接（多跳）的，由于图被分割以后本地只会存储直接的邻居，当需要访问间接邻居的时候就必须通过网络通信与其它存储节点进行交互，这里的网络通信代价在大规模图计算中是不容忽视的。解决思路也很直接，即在每个节点本地缓存顶点的间接邻居，但要缓存哪些顶点的邻居，要缓存几个邻居是需要仔细考量的问题。AliGraph 没有使用目前常见的一些缓存算法（如 LRU），而是提出了一种新的基于顶点重要性（importance）的算法来对间接邻居进行缓存。在有向图中计算一个顶点重要性的公式是 `入邻居的个数 / 出邻居的个数`，注意这里的邻居个数同样可以是直接的或者间接的。当这个公式的计算结果大于某个用户自定义的阈值时即认为这是一个「重要」的顶点。从实际测试中得出的经验值是通常只需要计算两跳（hop）的邻居个数就够了，而阈值本身不是一个特别敏感的数值，设置在 0.2 左右是对于缓存成本和效果一个比较好的平衡。选出所有重要的顶点以后，最终会在所有包含这些顶点的节点上缓存 _k_ 跳的出邻居（out-neighbor）。

GNN 算法通常可以总结为 3 个步骤：采样（sample）某个顶点的邻居，聚合（aggregate）这些采样后的顶点的 embedding，将聚合后的 embedding 与顶点自己的进行合并（combine）得到新的 embedding。这里可以看到采样是整个流程中的第一步，采样的效果也会直接影响后续计算的 embedding 结果。AliGraph 抽象了 3 类采样方法：遍历采样（traverse）、近邻采样（neighborhood）和负采样（negative）。遍历采样是从本地子图中获取数据；近邻采样对于 1 跳的邻居可以从本地存储中获取，多跳的邻居如果在缓存中就从缓存中获取否则就请求其它节点；负采样通常也是从本地挑选顶点，在某些特殊情况下有可能需要从其它节点挑选。

在采样完邻居顶点以后就是聚合这些顶点的 embedding，常用的聚合方法有：element-wise mean、max-pooling 和 LSTM。最后是将聚合后的 embedding 与顶点自己的进行合并，通常就是将这两个 embedding 进行求和。为了加速聚合和合并这两个算子的计算，AliGraph 应用了一个物化（materialization）中间向量的策略，即每个 mini-batch 中的所有顶点共享采样的顶点，同样的聚合和合并操作的中间结果也共享，这个策略会大幅降低计算成本。

在最后的评估环节用了两个来自淘宝的数据集，两个数据集之间只有大小的区别，大数据集是小数据集的 6 倍左右。大数据集的基础数据是：4.8 亿个用户顶点，968 万个商品顶点，65.8 亿条用户到商品的边，2.3 亿条商品到商品的边，用户平均有 27 个属性，商品平均有 32 个属性。当使用 200 个 worker（节点配置论文中没有说明）时大数据集只需要 5 分钟即可将整个图构建完毕，相比之下以往的一些方案可能需要耗费数小时。基于顶点重要性的缓存算法相比 LRU 这些传统算法也是明显更优。3 类采样方法的性能评估结果从几毫秒到几十毫秒不等，但最长也不超过 60 毫秒，并且采样性能与数据集大小不太相关。聚合和合并算子相比传统的实现也有一个数量级的性能提升，这主要得益于前面提到的物化策略。

AliGraph 目前已经开源（一部分？）但是换了一个名字叫做 [graph-learn](https://github.com/alibaba/graph-learn)，跟大多数深度学习框架一样，底层使用 C++ 语言实现并提供 Python 语言的 API，目前支持 TensorFlow，未来会支持 PyTorch。有意思的是刚刚开源不久就有人提了一个 [issue](https://github.com/alibaba/graph-learn/issues/16) 希望能够跟另外几个流行的 GNN 框架进行比较，但是项目成员的回答比较含糊。

## Building Uber’s Go Monorepo with Bazel

[[链接]](https://eng.uber.com/go-monorepo-bazel)

Uber 应该是除了 Google 以外很早选择在后端服务中大规模使用 Go 语言的公司之一，并贡献了很多著名的 Go 语言项目（如 [zap](https://github.com/uber-go/zap)、[Jaeger](https://github.com/jaegertracing/jaeger)）。早在 2017 年，Uber 的 Android 和 iOS 团队就已经只使用一个代码仓库进行开发，俗称 monorepo。实践 monorepo 最著名的公司应该还是 Google，有兴趣可以看看 [Why Google Stores Billions of Lines of Code in a Single Repository](https://research.google/pubs/pub45424) 这篇文章。现在后端团队也开始采用 monorepo 来管理 Go 语言项目，但是和客户端团队的不同之处在于没有用 [Buck](https://buck.build) 而是用 [Bazel](https://bazel.build)（前者是 Facebook 开源，后者是 Google 开源）。这篇文章介绍了在 monorepo 中将 Go 语言和 Bazel 结合遇到的一些问题。

## Optimising Docker Layers for Better Caching with Nix

[[链接]](https://grahamc.com/blog/nix-and-layered-docker-images)

恐怕大多数时候接触容器是从构建一个 Docker 镜像开始的，这一步往往也是最容易被忽视的。为什么我的镜像这么大？为什么每次拉取镜像都要从头开始？这些问题可能会随着使用时间越来越长逐渐浮现出来，要回答它们需要了解 Docker 镜像的一个核心概念「layer」，本质上你在 `Dockerfile` 里写的每一行命令都会生成一个 layer，一个镜像便是由很多 layer 构成。Layer 之间是有层级关系的，当拉取镜像时如果本地已经存在某个 layer 就不会重复拉取。在传统的 Linux 发行版中安装依赖时 Docker 是不知道具体有哪些文件被修改的，而 [Nix](https://github.com/NixOS/nix) 这个特殊的包管理器采用了不一样的设计思路使得安装依赖这件事情对于 Docker layer 缓存非常友好。衍生阅读推荐 Jérôme Petazzoni 写的关于如何减少镜像大小的[系列文章](https://www.ardanlabs.com/blog/2020/02/docker-images-part1-reducing-image-size.html)。

## Proposal: Permit embedding of interfaces with overlapping method sets

[[链接]](https://github.com/golang/proposal/blob/master/design/6977-overlapping-interfaces.md)

Interface 是 Go 语言一个重要的特性，类似很多其它语言中的概念，接口定义好以后是需要通过 struct 来实现的。但不同之处又在于 struct 不需要显式声明实现了什么 interface，只要满足 interface 中定义的接口就行，这个关键设计使得 Go 语言的 interface 使用场景可以非常灵活。跟 struct 一样 interface 也允许嵌套，也就是可以在一个 interface 定义中嵌套另一个 interface。如果同时嵌套了多个 interface，并且这些 interface 之间有重复的接口在编译时是会报错的。实际开发过程中为了规避这个限制可能需要修改 interface 的定义，这对于开发者来说不太友好。上面这个提案允许开发者在不修改代码的情况下避开这个限制，目前这个功能已经在 [Go 1.14](https://golang.org/doc/go1.14#language) 中发布。

## VexTab

[[链接]](https://github.com/0xfe/vextab)

不管是音乐创作还是音乐演奏，乐谱都是一个必不可少的东西。还记得刚学吉他那会儿非常热衷的一件事情就是去网上搜集各种歌曲的六线谱，这些乐谱的格式从最朴素的纯文本到高级的 [Guitar Pro](https://www.guitar-pro.com) 格式都有。再后来开始学习扒歌，也面临把扒下来的谱子纪录下来的需求。虽然 Guitar Pro 很好但毕竟是一个收费软件，文件格式也是私有的。就像我更喜欢 Markdown 而不是直接用 Word 一样，一直希望能有一个类似的标记语言用于编写乐谱。VexTab 即是这样一个专门用于编写五线谱和六线谱的语言，也提供一个 JavaScript 库方便嵌入到网页中。有意思的是 VexTab 的作者同时也是 Google 的一名员工。
