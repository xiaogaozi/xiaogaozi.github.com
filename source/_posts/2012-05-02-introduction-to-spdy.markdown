---
layout: post
title: "SPDY 简介"
date: 2011-06-11 11:26
comments: true
categories: Google
---

今天在看 CTF write-up 时发现[有人提到](http://michele.spagnuolo.me/articles/web-security/defcon-ctf19-quals-grab-bag-100-writeup.html) [SPDY](http://dev.chromium.org/spdy) 这样一个东西，貌似跟 Chrome 项目有关，于是在 Geek 原始冲动的驱使下了解了一下。

首先 SPDY 是一个应用层协议，它被创造出来的唯一目的就是让 Web 更快，更快，还是更快。Google 这家公司似乎很喜欢“快”这个东西，Chrome 从诞生到现在每次几乎必定宣传自己有多么得快，搞得大家已经产生了某种心理暗示。SPDY 诞生于 2009 年，其实这是对外公开发布的时间，开始研究的时间应该更早。众所周知，如今的 Web 是通过 HTTP 协议和 TCP 协议进行传输，但种种因素导致 HTTP 传输变得很慢：

- 每一个 TCP 连接一次只能发一个 HTTP 请求，这个估计是 HTTP 协议的最大弊端。想象一下如今的网站已经包含大量的图片、CSS、JS 需要加载，如果一个请求一个请求地发，那肯定会慢死，所以浏览器通常都是通过建立多个连接来回避这个问题，但毕竟治标不治本。
- 只能由客户端主动发起 HTTP 请求，即时有时服务器知道还需要回复其它资源，它也只能等客户端先发起再回复。服务器真可怜，太被动了。
- HTTP 头没有压缩，而且 HTTP 头也有一些冗余信息，比如 User-Agent 就没有必要每次都发来发去，太浪费带宽了。
- 数据压缩是可选的，Google 认为必须强制要求。

既然 HTTP 有这么多缺点，那应该不止 Google 自己想要解决，其实是有的，本着不重复造轮子的原则 Google 列举了现有的一些改进方案：

- [HTTP pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining)：以流水线的形式传输请求和数据，这里吐槽一下，以前在公司时 Facebook 的某牛来介绍时谈到了他们开发的 BigPipe，思想也是流水线，同样也是为了优化 Web 性能，不知道他们是不是借鉴了 HTTP pipelining，:)
- [SCTP](http://www.sctp.org/)：用于替代 TCP 的传输层协议，提供了 multiplexed streams（多路复用流）和 stream-aware congestion control（流感知拥塞控制）
- [SST](http://pdos.csail.mit.edu/uia/sst/)：同样用于替代 TCP 协议（TCP 同学真是众矢之的……），也可以运行在 UDP 协议之上。
- [MUX](http://www.w3.org/Protocols/MUX/) 和 [SMUX](http://www.w3.org/TR/WD-mux)：运行在传输层和应用层之间的中间协议，同样提供了复用流。

但是 Google 同学觉得以上这些都还不够，它要追求更大程度的性能提升。考虑到 TCP 现在应用还很广泛，想替代也不是一天两天的事情，但 HTTP 就不一样了，它是应用层的！所以说有自家的浏览器就是好办，发明个应用层协议马上就可以上线。SPDY 在刚出来的时候 Google 还在说这并不是用来替代 HTTP 协议的，它只是一个中间协议，但看看[最新的协议文档](http://dev.chromium.org/spdy/spdy-protocol)里面已经将 SPDY 分为了两层，其中一层被描述为 HTTP-like，大有取代 HTTP 的意图（Google 最近的[一篇文章](http://googledevelopers.blogspot.com/2012/05/spdy-performance-on-mobile-networks.html)已经直呼 SPDY 为“a replacement for HTTP”）。可以想到 Google 已经将提议提交给 IETF，也许未来的某一天我们就不再使用 HTTP 协议了。SPDY 主要有以下一些特性：

- multiplexed streams，一个 TCP 连接将支持无限的并发 HTTP 请求
- 请求优先级，因为现在支持并发请求，就必须得为每一个请求设置一定的优先级
- 压缩 HTTP 头，去掉多余的头信息
- 全部请求都是通过 SSL 加密，Google 认为安全网络连接必定是未来的发展方向，即使加密会微微增加一些传输时间
- Web 服务器将能够主动发起通信，也就是 server push
- 还有一个类似的叫 server hint，不同于 server push 的是它仅仅向客户端发送一个 suggest，提示客户端需要发送一个 HTTP 请求

这些改进到底能有多大提升？Google 给出的数据是 39%～55%，在丢包严重或高延迟环境下，SPDY 表现更加出色。

要支持 SPDY，除了客户端必须支持外，还要有相应的 Web 服务器。现在已经有 [Java](http://wiki.eclipse.org/Jetty/Feature/SPDY)、[Apache module](http://code.google.com/p/mod-spdy/)、[Python](http://github.com/mnot/nbhttp/tree/spdy)、[Ruby](https://github.com/igrigorik/spdy)、[node.js](https://github.com/indutny/node-spdy) 等各种实现。

最后，如果你正在使用 Chrome 浏览器，并且访问 Google 的网站，那你已经开始使用 SPDY 了，输入 [chrome://net-internals/#spdy](chrome://net-internals/#spdy) 还可以了解更加详细的信息。
