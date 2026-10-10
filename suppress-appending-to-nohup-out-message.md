---
title: "让nohup不输出 appending output to 'nohup.out'"
source: https://www.boris1993.com/suppress-appending-to-nohup-out-message.html
date: 2019-12-27
updated: 2026-10-10
tags: [Linux, nohup]
categories: [小技巧]
---

# 让nohup不输出 appending output to 'nohup.out'

在使用`nohup`的时候，它总会打印一条`nohup: appending output to 'nohup.out'`这样的信息，并且必须敲一下回车。

因为`nohup: appending output to 'nohup.out'`这条信息是打印到`STDERR`的，所以解决的方法很简单，把`STDERR`重定向至`STDOUT`就可以了，比如这样：

```bash
nohup doSomething > nohup.out 2>&1 &
```
