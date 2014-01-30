---
layout: post
title: "Auto Open Browser After Copy URL"
date: 2014-01-31 05:07:36 +0800
comments: true
categories: [AppleScript, Productivity]
---

前段时间看过一篇叫 [Automate Everyday Tasks](http://sc5.io/blog/2014/01/automate-everyday-tasks) 的博客，其中的一些见解很有意思，我们日常工作中有很多细小但是重复的事情，如果能够将某些工作自动完成，会让生活更加舒适。我很喜欢 Mac 上一个叫 [PopClip](http://pilotmoon.com/popclip) 的小 app，可以大大减少很多重复的操作。这篇博客就是介绍如何制作一个 app，当复制 URL 时自动在浏览器中打开。

<!-- more -->

有了这个想法之后我先去[找找看](https://www.google.com/search?q=os+x+clipboard+manager)是否有类似的软件，但已有的剪贴板管理工具都没有这样的功能。于是决定自己动手做，因为没有开发 Mac app 的经验，首先想到的就是利用 Automator 来实现，可惜 Automator 不支持后台运行。经过搜索 StackExchange 上的一个[问题](http://apple.stackexchange.com/questions/96214/creating-a-resident-workflow-with-automator)给了我思路：用 AppleScript 来做。

## AppleScript 程序

打开 AppleScript Editor，输入以下代码，代码大意是每隔 1 秒判断剪贴板内容是否为 URL，如果是就在浏览器中打开。

```applescript
property oldValue : missing value

on idle
    local newValue
    set newValue to the clipboard
    if oldValue is not equal to newValue then
        try
            if newValue starts with "http://" or newValue starts with "https://" then
                do shell script "open " & newValue
            end if
        end try
        set oldValue to newValue
    end if
    return 1
end idle
```

保存，「File Format」选「Application」，勾选「Stay open after run handler」。

{% img center http://f.cl.ly/items/3s0D1g2D2h2U1R273i1b/Screen%20Shot%202014-01-30%20at%2017.31.56.png %}

## 设置后台运行

AppleScript 程序运行时会在 Dock 上显示一个图标，我们需要隐藏这个图标。

{% img center http://f.cl.ly/items/343f2V1S2D3E1O102h0t/Screen%20Shot%202014-01-31%20at%200.57.07.png %}

{% img center http://cl.ly/image/2O0v2O23341w/Screen%20Shot%202014-01-31%20at%204.49.28.png %}

增加一个新的 key「Application is background only」，value 为「YES」。

{% img center http://f.cl.ly/items/0L3c0u1R47213D2b2F3N/Screen%20Shot%202014-01-30%20at%2017.58.09.png %}

## 设置登录自动启动

在 System Preferences → Users & Groups → Login Items 中添加刚才创建的 app，并设置为 hide 模式。

{% img center http://f.cl.ly/items/1t461t21143M1s1J1L1o/Screen%20Shot%202014-01-30%20at%2018.02.57.png %}
