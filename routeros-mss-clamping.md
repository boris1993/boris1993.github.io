---
title: "在RouterOS中配置MSS Clamping解决部分网站图片无法加载的问题"
source: https://www.boris1993.com/routeros-mss-clamping.html
date: 2023-05-14
updated: 2026-10-10
tags: [RouterOS, 二手MicroServer Gen8折腾记, Gen8, 软路由]
categories: [瞎折腾]
---

# 在RouterOS中配置MSS Clamping解决部分网站图片无法加载的问题

换成自己用RouterOS拨号之后，经常发现有的图片加载不出来，网上一顿冲浪之后发现可能是在RouterOS中没有正确配置MTU及MSS，导致部分包被丢弃，也就是传说中的PMTU黑洞。

## 症状

最常见的就是微信公众号的图片始终加载不出来，就像这样（自己当时没截图，借用知乎文章的图）：

![](https://blog-static.boris1993.com/routeros-mss-clamping/symptom.jpg)

说来也很奇怪，我在iOS的微信上是能正常加载的，但是换到我妈的Android微信就会出现这样的情况，之前一直以为是我妈的手机出毛病了……

## PMTU黑洞

所谓MTU，指的是一条链路上可以通过的三层数据包的最大尺寸（包含IP包头）。以太网默认的MTU是1500字节。但是从我的设备到目标服务器之间的路径上可能存在MTU小于1500的链路，那么这条路径上最小的MTU，就是整条链路的`Path MTU`（PMTU）。

路由器在转发包的时候，如果包的大小超过了MTU，那么这个包会被分片（fragmentation）。而终端设备在发包时，也可以设置DF标志位（Don’t Fragment）来告诉路由器不要对这个包分片，此时如果这个包大小超过了MTU，那么路由器就会丢掉这个包，并回复一条`ICMP Fragmentation Needed`消息。发送者收到这个消息后，下次就会发送小一点的包。这个过程叫做PMTU发现（PMTU Discovery）。

但是互联网中有大量的设备因为各种原因，会配置为不回应`ICMP Fragmentation Needed`消息，这使得大小超过MTU的包会被无声地丢掉，直到TCP协议发现超时丢包并进行重传。这种情况就是`PMTU黑洞`。

此外，IPv6包不支持分片，换句话说就是所有IPv6数据包全都带有DF标记。中间的路由器在遇到尺寸大于MTU的包的时候，应该回应`ICMPv6 Packet Too Big`消息，而同样的，由于各种原因，某些中间设备可能会直接丢掉这个包而不返回这条消息，直到TCP协议发现超时而进行重传。

## 为什么用光猫或者硬路由拨号就没有这个问题

这是因为，多数家用路由器默认开启了一个叫`MSS Clamping`的功能。这是针对PMTU黑洞的一个workaround，简单来说就是在TCP握手时，服务器会通过一个字段告知客户端它愿意接收的TCP包的最大尺寸，这样客户端就可以限制自己发送的包的大小，保证不会超出服务端要求的尺寸。

## 在RouterOS中配置MSS Clamping

配置非常简单，分别对IPv4和IPv6的防火墙Mangle表添加如下配置即可。

```routeros
# 将命令中 out-interface 参数的值替换成你的PPPoE接口的名字
/ip/firewall/mangle
add action=change-mss chain=forward comment="IPv4 MSS clamp to PMTU" \
    new-mss=clamp-to-pmtu out-interface="China Telecom" passthrough=yes \
    protocol=tcp tcp-flags=syn

/ipv6/firewall/mangle
add action=change-mss chain=forward comment="IPv6 MSS clamp to PMTU" \
    new-mss=clamp-to-pmtu out-interface="China Telecom" passthrough=yes \
    protocol=tcp tcp-flags=syn
```

## 参考文档

-   开启 IPv6 后网速变得很慢？可能是 PMTU 黑洞的问题 - V2EX
-   ROS修改MTU和MSS解决上网慢和页面显示不全问题
-   什么是最大分段大小 (MSS)？ - CloudFlare
