---
title: "配置RouterOS从内网访问光猫"
source: https://www.boris1993.com/access-to-fiber-optic-modem-through-lan.html
date: 2025-07-24
updated: 2026-10-10
tags: [光猫, 桥接, RouterOS]
categories: [瞎折腾]
---

# 配置RouterOS从内网访问光猫

我的宽带是自己换了光猫改成了桥接的，平时用起来没问题，只是我很不爽不能直接通过内网访问到光猫的配置页面。倒不是要经常改光猫配置，只是又犯了折腾病，想要把这一块缺失的拼图补上。一开始以为是配一条静态路由的事，但是发现并不奏效。上网学习了别人的成功经验之后，发现我的方向彻底错了，好在正确的操作也并不复杂。

因为我用的是搭载RouterOS的Mikrotik路由器，所以文中涉及的命令都将是RouterOS的语法。不过我也会讲明操作的原理，所以你也可以跟着思路针对你的路由器进行配置。唯一需要注意的就是，要确保你的路由器可以自己配置`iptables`规则。

如下是我的网络拓扑，其中：

-   光猫的管理地址为`192.168.0.1`，独占`192.168.0.0/24`网段
-   路由器的地址为`192.168.1.1`，它和内网的各个设备占用`192.168.1.0/24`网段

![](https://blog-static.boris1993.com/access-to-fiber-optic-modem-through-lan/network-topology.png)

那么要通过路由器访问到光猫的话，只需要做下面三件事：

-   给路由器的WAN口分配一个`192.168.0.0/24`网段的地址，比如`192.168.0.2`

```routeros
/ip/address/add address=192.168.0.2/24 comment="Modem access" interface=ether1 network=192.168.0.0
```

-   在防火墙的`nat`表中配置如下规则

```routeros
/ip/firewall/nat/add action=masquerade chain=srcnat comment="Access to modem subnet" dst-address=192.168.0.0/24 src-address=192.168.1.0/24
```

这条规则可以让路由器在收到向光猫发出的数据包后进行一次源地址伪装，使其看起来像是从路由器发出的，并在路由器收到响应的数据包后，再将其目标地址和端口修改会发出请求的电脑。

-   在防火墙的`mangle`表中配置如下规则

```routeros
/ip/firewall/mangle add action=accept chain=prerouting comment="Access to modem subnet" dst-address=192.168.0.0/24 src-address=192.168.1.0/24
```

这条规则将告知路由器在收到从内网访问光猫的数据包时，直接放行这个数据包，不再进行任何其他的标记或改变路由行为的操作。这条规则并不是必须的，但是如果你在`mangle`表中有其他的规则，那还是加上它比较好，以避免访问到光猫的数据包被错误的标记，或者产生非预期的行为。

上面两条RouterOS命令我也用ChatGPT翻译成了`iptables`命令，供参考：

```shell
# /ip firewall nat add action=masquerade chain=srcnat comment="Access to modem subnet" dst-address=192.168.0.0/24 src-address=192.168.1.0/24
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -d 192.168.0.0/24 -j MASQUERADE -m comment --comment "Access to modem subnet"

# /ip firewall/mangle add action=accept chain=prerouting comment="Access to modem subnet" dst-address=192.168.0.0/24 src-address=192.168.1.0/24
iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -d 192.168.0.0/24 -j ACCEPT -m comment --comment "Access to modem subnet"
```
