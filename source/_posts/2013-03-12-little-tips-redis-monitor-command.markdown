---
layout: post
title: "Little Tips: Redis MONITOR Command"
date: 2013-03-12 23:47
comments: true
categories: Redis
---

前段时间知乎的 cache 服务器中的某个数据总是错乱，想到了几个可能修改缓存的源头，同时在代码中搜索相关代码，把这些服务都重启了。但是问题依旧，只是没有之前那么严重。好吧，这下肯定是某个不知名的地方仍然在访问缓存。那就从根源查起，猛然发现 Redis 的 [MONITOR](http://redis.io/commands/monitor) 命令，可以实时打印出此时正在执行的命令，正合我意，修改缓存的命令我是知道的，只需要监测这个命令，然后就可以查到来源了。

    $ redis-cli monitor | grep '"set" "alist"'
