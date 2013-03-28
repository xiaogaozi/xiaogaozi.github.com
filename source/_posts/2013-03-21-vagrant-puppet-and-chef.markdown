---
layout: post
title: "流浪汉，木偶和厨子"
date: 2013-03-21 20:25
comments: true
categories: [Vagrant, Puppet, Chef]
---

最近要为 Phabricator 搭建虚拟测试环境，[Vagrant](http://www.vagrantup.com) 是一个不错的选择（话说官网现在更新以后，变得颇为华丽）。Vagrant 官方只[提供](https://github.com/mitchellh/vagrant/wiki/Available-Vagrant-Boxes) Ubuntu 的 base box，不过 [Vagrantbox.es](http://www.vagrantbox.es) 有提供很多其它的系统，甚至还有 Window$。也可以自己根据[官方文档](http://docs-v1.vagrantup.com/v1/docs/base_boxes.html)重新搭建一个 base box。

[Provisioning](http://docs.vagrantup.com/v2/provisioning/index.html) 是 Vagrant 一个很棒的特性，可以通过工具来自动配置和管理虚拟机。目前支持的有 Puppet 和 Chef，这两个都是著名的配置管理工具，其中 Google、Twitter、GitHub 在用 Puppet，Facebook 在用 Chef，知乎目前用的是 Puppet。正好这次两个都了解了一点，可以简单比较一下。

从安装方式来说，因为都是基于 Ruby 的工具，所以都可以通过 `gem` 来安装，从这一点上来说还是很方便的（话说对于 Mac 用户，千万别用官方提供的烂方法）。Puppet 的命令行工具就叫 `puppet`，而 Chef 的叫做 `knife`，这倒是跟 Chef 本身名字很搭。初学工具，肯定要看官方文档，在这一点上我觉得 Puppet 做得更好，至少还有一个像模像样的 [Learning Puppet](http://docs.puppetlabs.com/learning/) 系列，由浅入深，循序渐进，基本上看完就可以对 Puppet 有个大概的了解和使用。而 Chef 就只扔给你一个[不知道该从哪看起的页面](http://docs.opscode.com)，作为初学者表示很难入门。

Puppet 可以将一系列的配置文件打包成一个 module 供人下载，Chef 对应的则叫做 cookbook，这两者都提供了网站用于集中放置社区贡献的包，分别是 [Puppet Forge](http://forge.puppetlabs.com) 和 [Opscode Community](http://community.opscode.com/cookbooks)（不得不吐槽，这两个网站都很糙）。对于 module、cookbook 的安装及管理 Chef 略胜一筹，Puppet 的命令行工具可以很方便地安装 module，但是如果需要安装的包比较多，就只能通过自己写脚本来自动处理。而 Chef 有一个很好用的工具 [Librarian-Chef](https://github.com/applicationsonline/librarian)，只需要定义好所有依赖包，并放到 Cheffile 中，就可以通过 `librarian-chef` 命令来安装和管理。

Puppet 在易用性，社区质量和包的扩展性上来说要比 Chef 略优，能查到的文档资料也更多一点，最终我选择了 Puppet，[这里](https://github.com/xiaogaozi/vagrant-phabricator)是我的适用于 Phabricator 的配置文件，对 Chef 有兴趣的同学也可以看[这个](https://github.com/grigio/vagrant-chef-solo-bootstrap)示例配置。
