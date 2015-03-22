---
layout: post
title: "Octopress Rocks!"
date: 2011-10-23 17:12
comments: true
categories: [octopress, blog]
---

今天开始尝试 [Octopress](http://octopress.org/)，之前也有耳闻，当时立马被它的 Geek 气息吸引，今天详细了解，Octopress 真不愧为「A blogging framework for hackers」。我用了多年的 Blogger [博客](http://xiaogaozi.blogspot.com)也终于寿终正寝，如果你是一个非 hacker 博客作者，[Blogger](http://www.blogger.com/) 绝对值得推荐（满足 [GFW 三定律](http://www.dbanotes.net/review/gfw_rule.html)）。

## Another WordPress? ##

Yes。Octopress 具备一个博客应当具备的所有功能，文章、评论、页面、分享、RSS、搜索、Archives 等等。

No。正如 Octopress 网站介绍所说：A blogging framework for hackers，重点就在最后那个 hackers。没有了 WordPress 的后台界面，写博客需要的工具仅仅是 Ruby、Git、Markdown 和你喜爱的编辑器。如果你是一个 hacker，那你对这些工具不会陌生，相比 WordPress 蹩脚的后台页面，Octopress 提供的写作方式会让你非常喜爱。

## 写作 ##

博客最重要的功能就是写作，写作就像程序员编写代码，如果不能提供舒服的方式，那简直是一种自虐。事实已经证明 HTML 不是一种好的写作方式，因此 WordPress 这类博客提供了所见即所得编辑器，但对于喜欢精确掌控的 hacker 来说这还不够，于是类似于 Markdown 这样的标记语言逐渐在圈内盛行。这类标记语言最大的好处就是让作者不用关心文章的样式，而专注于文章的内容。这很重要，一篇文章的精髓在于文字，如果过多地被样式困扰，精力便会分散，也必然不会思考出更好的文字。类似的比较还有 Word 和 LaTeX，当然这种观点也是仁者见仁，智者见智。

Octopress 原生为我们提供了 Markdown 支持，需要写一篇新博客了？打开你喜欢的编辑器，使用 Markdown 语法开始书写即可。WordPress？虽然也可以添加 Markdown 支持，但不免显得蹩脚。

## 语法高亮 ##

这个功能对程序员来说尤为重要，但至今没有博客提供原生支持，这也是最大的遗憾。Octopress 彻底颠覆了这种局面，语法高亮变得如此顺其自然。

{% codeblock rename.py https://raw.github.com/xiaogaozi/princess-alist/master/home/xiaogaozi/src/Python/rename.py download %}
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# Batch rename utility. <https://github.com/xiaogaozi/princess-alist>
# Copyright (C) <2011>  xiaogaozi <gaochangjian@gmail.com>
# 
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
# 
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
# 
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

import os
import re
import sys

def usage():
    """Usage"""
    print "Usage: rename.py expr files"

def main():
    """Main progress."""
    if len(sys.argv) < 3:
        usage()
        sys.exit(1)

    expr = sys.argv[1]
    m = re.search(r'^[sy]/([^/]*)/([^/]*)/$', expr)
    if m is None:
        sys.stderr.write("expression incorrect\n")
        sys.exit(1)
    re1 = m.group(1)
    re2 = m.group(2)  # actually in substitute mode this portion is not regular expression

    for oldfile in sys.argv[2:]:
        d = os.path.dirname(oldfile)
        oldname = os.path.basename(oldfile)
        newname = ''
        newfile = ''
        if expr[0] == 's':  # substitute
            newname = re.sub(re1, re2, oldname)
            newfile = os.path.join(d, newname)
            os.rename(oldfile, newfile)
        elif expr[0] == 'y':  # transliterate
            pass
        print oldfile, '->', newfile

if __name__ == "__main__":
    main()
{% endcodeblock %}

## 版本控制 ##

程序员已经被版本控制惯坏了，只要能纳入版本库，就统统放进去。WordPress 拥有同样蹩脚的版本控制，显然不足以满足 hacker 的需求，Octopress 为我们提供了 Git 原生支持，一切一切都为你所控，放在你喜欢的版本控制库里即可。

## 部署 ##

Octopress 为我们提供了三种部署方式：GitHub Pages，Heroku，Rsync，在我看来，其实就两种：免费和收费。GitHub Pages 和 Heroku 都是免费使用，Rsync 则需要你拥有自己的虚拟主机。<del>我选择了 Heroku，毕竟 GitHub Pages 本意是用来放项目介绍页面的，结果被强大的 hacker 们发掘来作为博客了⋯⋯</del> 我现在使用的是 GitHub Pages。

## Continue? ##

开始享受写作的乐趣吧～
