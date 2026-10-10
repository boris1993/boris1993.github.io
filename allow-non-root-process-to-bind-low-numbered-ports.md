---
title: "如何允许非root进程绑定低位端口"
source: https://www.boris1993.com/allow-non-root-process-to-bind-low-numbered-ports.html
date: 2018-08-23
updated: 2026-10-10
tags: [setcap, low numbered ports]
categories: [小技巧]
---

# 如何允许非root进程绑定低位端口

众所周知，在Linux中，要想绑定端口号在1024以下的低位端口，是需要root权限的。但是，如果我又想绑定低位端口，又不想以root身份运行程序，该怎么办？答案是，`setcap`命令。

# 怎么做

TL;DR，使用如下命令给这个程序赋予`CAP_NET_BIND_SERVICE`能力即可。注意，这条命令需要以root身份执行。

```bash
sudo setcap CAP_NET_BIND_SERVICE=+eip /path/to/binary
```

# 这到底在干嘛

接下来，我就把上面这条命令一点点拆开，解释清楚它到底干了什么。

## Capabilities

首先介绍一下`capabilities`(能力)这个东西。

在Linux内核版本2.2开始，Linux将一系列的超级管理员权限细分成了一个个可以单独开启关闭的单元，以提供更细粒度的权限控制，这些单元，就被称之为`capabilities`。详细的`capabilities`列表可以参考[Man Page Capabilities(7)](#ref_1)。

## CAP\_NET\_BIND\_SERVICE

拥有这个`capability`的程序，就可以绑定端口号在1024以下的特权端口。

## setcap

那么，该如何控制每个`capability`呢？答案就是`setcap`命令。上文所提到的命令，就是给指定的这个二进制程序增加`CAP_NET_BIND_SERVICE`这个`capability`。

在`capability`名后面，用加号相连接的，则是开启这个`capability`的模式。模式有如下三种：

-   e: Effective，意为这个`capability`是启用的。
-   p: Permitted，意为这个`capability`是允许被使用的。
-   i: inherited，意为这个`capability`可以被其子进程继承。

在`setcap`命令中，使用加号来开启这个模式，或者使用减号来关闭这个模式。

# 有什么副作用

这个方法确实有一些副作用，或者说是限制：

1.  这个方法对脚本无效。如果要使某个脚本拥有这个能力，则需要为其解释器赋予这个能力，而这明显是一个巨大的安全隐患。
2.  Linux会为使用了`setcap`或`suid`的程序禁用掉`LD_LIBRARY_PATH`。

# 除了手动指定，还有没有其他办法

`Systemd`也支持在`service`的配置文件中指定`capabilities`，其用法示例如下：

```ini
[Service]

# 该服务仅可以使用哪些capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

# 以非特权用户运行程序时需要设定此参数
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

# 参考资料

1.  Man Page Capabilities(7)
3.  Man Page setcap(8)
5.  Man Page cap\_from\_text(3)
7.  getcap, setcap and file capabilities - insecure.ws
9.  Is there a way for non-root processes to bind to “privileged” ports on Linux? - Stack Overflow
11.  Linux的capabilities机制
