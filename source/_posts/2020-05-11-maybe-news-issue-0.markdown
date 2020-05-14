---
layout: post
title: "Maybe News Issue #0"
date: 2020-05-11 14:44:52 +0800
comments: true
categories: [maybe news]
---

> 前言：从这一期开始这个系列将会有一个正式的名字「Maybe News」，名字来源于我非常喜欢的一个国内的音乐厂牌[「兵马司」](https://en.wikipedia.org/wiki/Maybe_Mars)（Maybe Mars）。本身我分享的内容也很有可能是一些旧闻，只不过对于我来说是还未了解的知识罢了。[上一期](/2020/04/26/weekly-reading-list-issue-1/)的名字还是维持原样就不做修改。你也可以通过[邮件订阅](https://digest.xiaogaozi.org/maybe-news)这个系列的文章。

<!-- more -->

## LightRec: a Memory and Search-Efficient Recommender System

[[链接]](http://staff.ustc.edu.cn/~liandefu/paper/lightrec.pdf)

这篇论文由微软亚洲研究院与中科大共同发表在 [WWW 2020](https://www2020.thewebconf.org) 会议上，提出了一种新的表示物品向量的方法，大幅降低存储向量所需空间的同时还显著提升了召回效果。一个直观的数据：LightRec 将 1 千亿 256 维双精度向量的内存占用从 9.5 GB 降到了 337 MB，这是非常惊人的！现在工业界常用的 [nmslib](https://github.com/nmslib/nmslib) 和 [Faiss](https://github.com/facebookresearch/faiss) 都无法实现如此高的压缩比，因此很多时候都需要借助分布式存储来满足业务场景，如果真的如论文中所描述的一样那单机存储在未来很长一段时间来说都是完全足够的。

这里简单解释一下为什么向量召回对于当下的推荐系统如此重要，传统的召回是基于倒排索引的方式，正如我在[之前的一篇文章](/2020/04/21/how-to-design-a-distributed-index-framework-part-1/)中介绍的那样，召回与模型优化目标之间的差异较大导致召回效果始终较差。自从 [Learning Deep Structured Semantic Models for Web Search using Clickthrough Data](https://www.microsoft.com/en-us/research/publication/learning-deep-structured-semantic-models-for-web-search-using-clickthrough-data) 这篇论文（同样也是由微软研究院发表）提出 DSSM（Deep Structured Semantic Models）以后，将召回与 DNN 进行结合，显著提升了召回的效果，在很多公司的实践中也的确论证了 DSSM 是一个非常有效的召回方式。DSSM 的核心是分别为物品和用户生成向量，再通过 ANN（Approximate Nearest Neighbors）查询相似向量从而实现召回。因此向量的存储和查询效率决定了在线请求的效果和性能，如何平衡向量索引的空间占用和召回效果是非常重要的。

微软研究院的微信公众号有一篇简短的针对这篇论文的[中文版介绍](https://mp.weixin.qq.com/s/E43gc16A3OVWgxyfdUxr7g)，有兴趣也可以先看这篇文章。

## TFRT: A new TensorFlow runtime

[[链接]](https://blog.tensorflow.org/2020/04/tfrt-new-tensorflow-runtime.html)

Google 近期开源了新的 TensorFlow 运行时 TFRT（TensorFlow Runtime），这是一个介于上层用户代码和底层设备之间的执行环境。项目的愿景是实现一个统一的、可扩展的、性能首屈一指（best-in-class）的，同时可跨越多种领域硬件（domain specific hardware）的运行时。未来 TFRT 会成为 TensorFlow 默认的运行时，目前还在集成中。从 ResNet-50 的 inference 测试结果上看平均提升了 28% 的性能。

## Why We Need DevOps for ML Data

[[链接]](https://tecton.ai/blog/devops-ml-data)

虽然这是一篇产品推广软文（在文章最后一节），但是文章中普及的关于 DevOps 与机器学习之间的关系还是非常有价值的。很多人可能以为机器学习就只是模型算法而已，诚然这是学术研究的基石，但是要真正把机器学习应用到工业界光有算法是远远不够的。Google 著名的 [Hidden Technical Debt in Machine Learning Systems](https://papers.nips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems.pdf.) 论文已经论述了那些隐藏在模型背后的往往被人忽略的技术，模型规模越大需要付出的工程努力也是越大的（所以很多时候大公司才需要自己造轮子）。作为衍生阅读也可以同时看看 [How LinkedIn, Uber, Lyft, Airbnb and Netflix are Solving Data Management and Discovery for Machine Learning Solutions](https://towardsdatascience.com/how-linkedin-uber-lyft-airbnb-and-netflix-are-solving-data-management-and-discovery-for-machine-9b79ee9184bb) 这篇文章。

## Mid-stack inlining in Go

[[链接]](https://dave.cheney.net/2020/05/02/mid-stack-inlining-in-go)

Dave Cheney 继续科普 Go 的一些实现细节，这次的主题是编译器如何实现 mid-stack inlining。所谓 mid-stack inlining 就是将那些调用了其它函数的函数变成 inline，相对的还有 leaf inlining，即不调用任何其它函数。有兴趣了解 leaf inlining 的可以看 Dave Cheney 的[上一篇文章](https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go)。

## Why We Leverage Multi-tenancy in Uber’s Microservice Architecture

[[链接]](https://eng.uber.com/multitenancy-microservice-architecture)

Uber 介绍了他们在微服务领域实践的一个经验「多租户」，简单讲就是让请求链路上的所有组件和系统都能够感知「租户」这个概念，比如租户可以分为生产环境和测试环境。Uber 列举了两个应用场景：集成测试和 Canary 部署，这两个场景都依赖生产环境的请求，有了租户的概念就可以自动进行请求路由和数据隔离。愿景其实挺美好，但「代价」也是不容忽视，前面讲了要让所有组件和系统都感知就非常依赖基础组件的统一，要解决这个问题很多时候并不单纯是一个技术问题。如何做好不同环境的数据隔离也是一个难题，关于这一点文章并没有做特别详细的介绍。
