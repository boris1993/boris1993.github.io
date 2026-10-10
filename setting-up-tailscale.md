---
title: "使用Tailscale部署虚拟局域网"
source: https://www.boris1993.com/setting-up-tailscale.html
date: 2022-10-24
updated: 2026-10-10
tags: [Tailscale, 内网穿透, VPN, Virtual LAN, 虚拟局域网]
categories: [瞎折腾]
---

# 使用Tailscale部署虚拟局域网

最近折腾了下用Tailscale搭建虚拟局域网，在这里记录一下折腾的过程和一些心得。

## 什么是Tailscale

说起Tailscale，就不得不提到最近很火的Mesh VPN这个概念。

经典的VPN是作为一个网关，让外网用户通过它来访问内网。比如传统的L2TP VPN：

![](https://blog-static.boris1993.com/setting-up-tailscale/network-topology-with-l2tp-vpn.png)

但是Mesh VPN提出了一种全新的拓扑结构，这种结构中每个节点之间都相互连接，不再存在中心的VPN服务器。

![](https://blog-static.boris1993.com/setting-up-tailscale/mesh-vpn-with-wireguard.png)

上图提到的`Wireguard`就是一个Mesh VPN的实现，但是`Wireguard`配置比较繁琐，维护成本也比较高，所以`Tailscale`在基于`Wireguard`的基础上做了一些优化，比如用于NAT穿透的`DERP`协议、建立了公网coordinator服务器用于下发配置，以及集成了第三方SSO用于认证和权限管理。

## 如何安装和使用Tailscale

安装Tailscale很简单（除了国内群晖），跟着官网的Quickstart走就行了。简单来说就是先在官网注册账号，然后在电脑上安装Tailscale并使用相同账号登陆，然后这些机器就处于同一个虚拟局域网上了。

登陆的时候注意，千万不要用GitHub或者Google认证，原因我不用说，你懂的。就算电脑上你有办法，手机上也会很闹心。

在机器成功加入Tailscale虚拟局域网后，就可以在Tailscale控制面板里面看到这些机器的IP地址了。

![](https://blog-static.boris1993.com/setting-up-tailscale/tailscale-admin-portal.png)

这时候，就可以直接使用这些IP地址访问到对应的机器了。

### 国内群晖安装Tailscale

因为国内群晖的套件商店下架了所有VPN相关的套件，所以只能到官网的Tailscale Packages - stable track手动下载对应DSM版本的spk，然后到套件中心手动安装。安装成功后就可以在主菜单中看到Tailscale的图标，点击启动并在新页面中使用相同账号登陆，然后就能用了。

## Tailnet和Magic DNS

当然Tailscale并不仅支持通过IP地址访问机器，还可以通过域名访问。这里请放心，Tailnet的域名解析是通过你本机的Tailnet客户端解析的。

要启用这个功能，首先你需要选择一个Tailnet域名。Tailscale会给你一系列域名让你选，选好之后就会在控制面板的DNS页面出现。

![](https://blog-static.boris1993.com/setting-up-tailscale/tailscale-dns-tailnet.png)

然后，在`Magic DNS`栏目下面点`Enable Magic DNS`就可以了。

![](https://blog-static.boris1993.com/setting-up-tailscale/tailscale-dns-magic-dns.png)

在这之后，就可以直接用控制面板中的设备名，或者`设备名.Tailnet域名`的方式访问你的设备了。比如我要SSH到`ds218plus`这台机器上，我就可以`ssh ds218plus`或`ssh ds218plus.tailnet-a8cf.ts.net`。

## 访问内网机器很慢

在我试用了几天之后，我发现了一个问题，如果节点双方都有IPv6的话，互联速度很快，但是如果节点一方没有IPv6的话，速度就出奇的慢。遂用`tailscale status`命令看了下连接情况，不出意外，看到了这样的返回：

```plaintext
$ tailscale status
100.98.89.5     macbook-pro          boris1993@   macOS   -
100.88.234.12   boris-iphone         boris1993@   iOS     offline
100.105.198.123 ds218plus            boris1993@   linux   active; relay "sfo", tx 1778904 rx 63968652
```

划重点，`relay "sfo"`，这说明我连接到`ds218plus`的通信，是经过位于旧金山的`DERP`服务器中转的，所以会慢。

虽然在What firewall ports should I open to use Tailscale?里面说，

> Nearly all of the time, you don’t need to open any firewall ports for Tailscale. Tailscale uses various NAT traversal techniques to safely connect to other Tailscale nodes without manual intervention—it “just works.”

也没错，它确实work，但是我真的没法说它work的well。好在，文章下面提到，除了`443/tcp`之外，Tailscale还用到`41641/udp`来让Wireguard节点互联，以及`3478/udp`用于STUN协议。

在我的这个使用场景中，`ds218plus`作为服务器，那么我就在`ds218plus`所在的网络中做了两件事，给`41641/udp`配置端口转发，并在防火墙上放行这个端口。然后我在作为客户端的`macbook-pro`上执行`tailscale down`和`tailscale up`来重启Tailscale连接。这时候再用`tailscale status`命令检查连接情况，发现输出变成了这样：

```plaintext
$ tailscale status
100.98.89.5     macbook-pro          boris1993@   macOS   -
100.88.234.12   boris-iphone         boris1993@   iOS     offline
100.105.198.123 ds218plus            boris1993@   linux   active; direct 121.70.200.131:41641, tx 5884 rx 8500
```

接着划重点，`direct`，说明现在节点之间已经是直连了，随便做点什么测试一下，果然快了很多。

不过端口转发只是权宜之计，毕竟没法同时让`41641/udp`转发给多个机器，只能解决一台机器不能直连的问题。要想彻底解决，还是得用IPv6。
