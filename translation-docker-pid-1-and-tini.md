---
title: "Docker中的PID 1和tini：为什么你的容器不响应Ctrl-C"
source: https://www.boris1993.com/translation-docker-pid-1-and-tini.html
date: 2025-06-07
updated: 2026-10-10
tags: [Docker, tini]
categories: [翻译]
---

# Docker中的PID 1和tini：为什么你的容器不响应Ctrl-C

之前我写过一篇文，讲我是怎么[处置Docker容器产生的僵尸进程的](https://www.boris1993.com/finding-the-docker-container-of-a-zombie-process.html)，正巧前两天上网乱刷，看到有个人也被容器中的僵尸进程困扰，有一条回复提到了一个关键词`tini`，说能根治这个问题，于是继续上网冲浪，翻到了Medium上的这篇文章，感觉很有用，所以翻译出来。

以下内容除特别注明外，皆翻译自原文。我亦不对内容做任何的担保，并不对任何可能产生的后果（包括但不限于文件丢失）负责。

在使用Docker的时候，你有可能会遇到这么一种很难受的情况，就是你敲了Ctrl-C想停掉这个容器，但这个容器却无动于衷。或者又可能你的容器停止了，但留下了一堆僵尸进程。这些问题通常来自于一个开发者们从一开始就没想明白的问题 —— 如果你的程序成为了容器中的PID 1会怎么样。

## 什么是PID 1

在Linux系统中，PID 1（进程号1）是在系统启动过程中第一个启动的进程。它扮演着一个特殊的角色，即系统的初始化者。它将负责启动和管理所有其他的进程。在Docker容器中，你启动的第一个进程将默认成为PID 1。

例如：

```shell
docker run -it node:18 node
```

这时候，Node.js的进程就是PID 1。

## 为什么PID 1很特别？

PID 1的进程在Linux中会有如下几种特殊的行为：

1.  响应信号的方式不同  
    大多数UNIX进程会自动接收并处理类似`SIGINT`（来自Ctrl-C）和`SIGTERM`（来自`docker stop`命令）的信号。  
    但是PID 1的进程默认不会接收这些信号，除非它们主动监听。
2.  收割僵尸进程  
    如果PID 1不等待子进程，那么这些子进程就会变成僵尸。尽管它们已经退出了，但仍然会消耗系统资源。（译者注：我理解就是在容器停止的时候，主进程不等待它的子进程全部退出成功再退出，而是就自己拍拍屁股走人了）  
    这会慢慢地拖慢这个容器的性能，或让这个容器的行为变得失控。

## 问题示例

假设我们有这样一个简单的Node.js应用：

```javascript
// app.js
setInterval(() => console.log('Running...'), 1000);
```

这个应用会运行在一个Docker容器中：

```shell
docker run -it node:18 node app.js
```

如果此时你尝试使用Ctrl-C退出，那么什么都不会发生。很奇怪对吧？因为：

-   Node.js在容器中是PID 1进程
-   当它是PID 1的时候，它没有正确转发或监听`SIGINT`信号

最终，你可能会一边纳闷为啥Ctrl-C不好使，一边反复敲它。这时候就是`tini`出场的时候了。

## tini：轻量的init

`tini`是个简化的初始化（init）系统，体积只有几KB，并且专为容器环境设计。Docker甚至集成了它，你只需要用`--init`参数就能开启。

## tini负责干什么？

1.  信号转发  
    它会监听类似`SIGINT`和`SIGTERM`之类的信号，并正确地将其转发到你的应用程序。这样一来，Ctrl-C或`docker stop`就会按照预期工作了。
2.  收割僵尸进程  
    `tini`会收割死亡的子进程，这样它们就不会变成僵尸了。
3.  就像一个负责任的PID 1一样干活  
    它工作起来就像一个真正的Linux初始化系统，只是更小了点。

## 怎么用tini？

### 方法1:使用Docker内置的功能

Docker在引擎内部已经集成了`tini`，所以你在运行容器时加上`--init`参数，那么Docker就会自动使用`tini`作为容器内的PID 1进程。你不需要额外安装或配置什么东西，只需要在命令中加上`--init`，就像这样：

```shell
docker run --init -it node:18 node app.js
```

### 方法2:在Dockerfile中手动添加tini

你也可以显式地为`Dockerfile`添加`tini`：

```dockerfile
FROM node:18

# 安装tini
RUN apt-get update && apt-get install -y tini

# 使tini成为入口点
ENTRYPOINT ["/usr/bin/tini", "--"]

# 运行你的应用
CMD ["node", "app.js"]
```

然后你可以像往常一样构建和运行它：

```shell
docker build -t node-app .
docker run -it node-app
```

## 为什么在生产环境会很重要？

在生产环境（比如Kubernetes）中，无法处理信号可能会引发这些问题：

-   应用程序无法干净的退出
-   数据因为处理停机的逻辑没有触发而导致损坏
-   僵尸进程造成资源泄漏

使用`tini`就可以用最小的成本避免这些问题。

## 最后的一点想法

尽管容器内部的行为很容易被忽略，但理解PID 1如何工作，以及`tini`解决了什么问题，能让你的容器变得更加干净、安全，并更容易维护。所以下次你遇到哪个应用不响应Ctrl-C，记得叫来小小的`tini`。

* * *

译者注：如果你使用Docker Compose编排容器的话，那么在配置文件中指定`init: true`就可以引入`tini`了，就像这样：

```yaml
services:
    app:
        image: alpine:latest
        init: true
```
