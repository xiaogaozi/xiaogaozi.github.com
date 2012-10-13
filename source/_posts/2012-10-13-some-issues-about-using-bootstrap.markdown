---
layout: post
title: "使用 Bootstrap 的几个问题"
date: 2012-10-13 21:24
comments: true
categories: [Bootstrap, JavaScript, CSS]
---

## Responsive 与 Modal ##

在开启 responsive 后，小屏幕设备上显示 modal 时会变成一闪而过，然后浮动窗口就不见了。具体效果可以缩小浏览器尺寸，在[这个页面](http://twitter.github.com/bootstrap/javascript.html#modals)的 Live demo 点击「Launch demo modal」看到。[Issue #2130](https://github.com/twitter/bootstrap/issues/2130) 专门讨论了这个问题，目前比较好的解决办法是使用[这个插件](http://niftylettuce.github.com/twitter-bootstrap-jquery-plugins)，根据页面大小来动态调整 modal 的位置，不过貌似用了之后 modal 那个由上至下显示的动画就没有了。这个 issue 现在还处于开启状态，看来官方短期内是不会解决这个问题的。

## Responsive 与 Navbar ##

responsive 模式下的 navbar 显示效果很赞，但是有一个很令人费解的事情，默认情况下所有 dropdown menu 都是展开的，对于使用多个菜单项，且子菜单条目很多的场景这是不能接受的。于是 [Issue #3184](https://github.com/twitter/bootstrap/issues/3184) 出现了，这次的方案比较 hack，需要修改 bootstrap-responsive.css，将 `.nav-collapse .dropdown-menu` 里的 `display: block;` 注释掉。这时你会惊喜地发现 dropdown menu 默认折叠了，点击也能展开子菜单。但是（总是有很多但是），在触屏设备上子菜单是选不中的！托 [filod](http://www.filod.net) 同学的福，修改 bootstrap-dropdown.js 中的一段代码：

{% codeblock lang:javascript %}
/* APPLY TO STANDARD DROPDOWN ELEMENTS
 * =================================== */

$(function () {
  $('html')
    .on('click.dropdown.data-api touchstart.dropdown.data-api', clearMenus)
  $('body')
    .on('click.dropdown touchstart.dropdown.data-api', '.dropdown form', function (e) { e.stopPropagation() })
    .on('click.dropdown.data-api touchstart.dropdown.data-api'  , toggle, Dropdown.prototype.toggle)
    .on('keydown.dropdown.data-api touchstart.dropdown.data-api', toggle + ', [role=menu]' , Dropdown.prototype.keydown)
})
{% endcodeblock %}

这里同时监听了 click 和 touchstart 事件，于是在触屏设备上先有 touchstart 将子菜单隐藏，再有 click 点击到隐藏后该位置的菜单项，因此你永远都不可能点到想点的子菜单。根本原因也是因为我们之前注释了 `display: block;` 引起，改变了 Bootstrap 的使用场景，于是 JS 出现如此纰漏。解决方法便是不监听 touchstart 事件，虽然会造成些小问题，不过也算基本满足要求。这个 issue 官方明确[表示](https://github.com/twitter/bootstrap/issues/3184#issuecomment-8072507)不会采纳，不过还是希望以后有机会增加一个开关选项给用户。
