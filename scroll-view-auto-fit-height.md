---
title: "微信小程序scroll-view填满剩余可用高度"
source: https://www.boris1993.com/scroll-view-auto-fit-height.html
date: 2018-08-10
updated: 2026-10-10
tags: [小程序, scroll-view]
categories: [学知识]
---

# 微信小程序scroll-view填满剩余可用高度

根据微信小程序scroll-view文档所述，`scroll-view`必须给定一个固定高度。那么如果我们想要让它自动填充剩余高度，该怎么办呢？

# 前言

在说出我的解决方案之前，先来看一下我的页面设计，以便于理解。

![Page Design](https://blog-static.boris1993.com/scroll-view/page-design.png)

如图所示，我将这个页面分成了三部分：最顶部的导航栏`navbar`，用于显示概要信息的`header`，以及本文的主角`scroll-view`。可见，`scroll-view`位于页面的最下方，如果我直接给它设定一个固定的高度，那么在不同尺寸的屏幕上，就可能会有高度过小而在下方留白，或者高度过大超出屏幕下边界的可能。那么，自动计算`scroll-view`的高度，看起来是一个可行的办法。

思路有了，接下来就开始挑趁手的工具吧！

# 需要的API

首先，在计算过程中，整个页面的高度是必须要有的。而小程序的wx.getSystemInfo API正好可以提供这样的功能。

其次，我们还得想办法拿到`scroll-view`上面各个组件的高度。小程序虽然没有DOM操作，但也提供WXML节点信息的API。

# 撸起袖子开始干

既然工具有了，那么，talk is cheap, I’ll show you the code!

当然，简洁起见，我只会写出相关的代码，其余的代码我将直接略掉。

```javascript
Page({
    data: {
        // 页面总高度将会放在这里
        windowHeight: 0,
        // navbar的高度
        navbarHeight: 0,
        // header的高度
        headerHeight: 0,
        // scroll-view的高度
        scrollViewHeight: 0
    },
    onLoad: function(option) {

        // 先取出页面高度 windowHeight
        wx.getSystemInfo({
            success: function(res) {
                that.setData({
                    windowHeight: res.windowHeight
                });
            }
        });

        // 然后取出navbar和header的高度
        // 根据文档，先创建一个SelectorQuery对象实例
        let query = wx.createSelectorQuery().in(this);
        // 然后逐个取出navbar和header的节点信息
        // 选择器的语法与jQuery语法相同
        query.select('#navbar').boundingClientRect();
        query.select('#header').boundingClientRect();

        // 执行上面所指定的请求，结果会按照顺序存放于一个数组中，在callback的第一个参数中返回
        query.exec((res) => {
            // 分别取出navbar和header的高度
            let navbarHeight = res[0].height;
            let headerHeight = res[1].height;

            // 然后就是做个减法
            let scrollViewHeight = this.data.windowHeight - navbarHeight - headerHeight;

            // 算出来之后存到data对象里面
            this.setData({
                scrollViewHeight: scrollViewHeight
            });
        });
    }
})
```

至于WXML里面，就还是使用双大括号来将`data`部分的`scrollViewHeight`的值绑定到`height`属性上面就是了。

需要注意的是，上面计算出来的值，单位是`px`而不是`rpx`。

```xml
<scroll-view style="height: {{scrollViewHeight}}px" scroll-y="true">
  <!-- scroll-view里面的内容 -->
</scroll-view>
```

这样，我们就得到了一个可以自动填满屏幕最下方剩余空间的`scroll-view`啦～
