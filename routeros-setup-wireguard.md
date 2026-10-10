---
title: "RouterOS配置Wireguard与阿里云ECS组网"
source: https://www.boris1993.com/routeros-setup-wireguard.html
date: 2023-11-10
updated: 2026-10-10
tags: [RouterOS, Wireguard]
categories: [瞎折腾]
---

# RouterOS配置Wireguard与阿里云ECS组网

前两天赶上阿里云搞活动，薅了一个99一年的ECS。因为要接进咱的探针，就得跟我的服务器组个网。之前都是用NAS上装的`wgeasy`，这回想想，咱的RouterOS也有Wireguard，那干脆吃个螃蟹，看看能不能直接在软路由上组，慢慢的逐步把`wgeasy`淘汰掉，顺便正经的学学Wireguard组网。

## 网络规划

我的LAN网段：`192.168.1.0/24`  
Wireguard网段：`10.10.0.0/24`  
分配给ECS的Wireguard地址：`10.10.0.100/32`

## RouterOS端的准备

首先添加一个`Wireguard`接口：

```routeros
/interface/wireguard/add disabled=no listen-port=13231 mtu=1420 name=wireguard1
```

然后添加防火墙规则，允许`13231/udp`端口：

```routeros
# in-interface更换成你的WAN接口的名字
/ip/firewall/filter add chain=input in-interface="China Telecom" dst-port=13231 protocol=udp action=accept comment="Allow Wireguard"
```

然后为Wireguard分配一个网段：

```routeros
/ip/address add address=10.10.0.0/24 network=10.10.0.0 interface=wireguard1 comment=Wireguard
```

接下来配置转发规则，允许LAN和Wireguard网段互通：

```routeros
/ip/firewall/filter add chain=forward src-address=192.168.1.0/24 dst-address=10.10.0.0/24 action=accept comment="Forward Wireguard to LAN"
/ip/firewall/filter add chain=forward src-address=10.10.0.0/24 dst-address=192.168.1.0/24 action=accept comment="Forward LAN to Wireguard"
```

至此，RouterOS上的Wireguard接口配置完毕。接下来需要取得RouterOS端Wireguard的公钥备用。

```routeros
# 记下输出中的 public-key 备用
> /interface/wireguard/print
Flags: X - disabled; R - running 
 0  R name="wireguard1" mtu=1420 listen-port=13231 private-key="routeros-private-key" public-key="routeros-public-key" 
```

## ECS端的准备

接下来登陆到ECS，安装Wireguard。

```bash
# 安装Wireguard
sudo apt install wireguard-tools

# 生成密钥对
mkdir wireguard && cd wireguard
wg genkey | tee privatekey | wg pubkey > publickey

# 查看公钥和私钥，备用
cat privatekey
cat publickey
```

## 开始组网

先回到RouterOS，为Wireguard添加一个Peer：

```routeros
# allowed-address为我们要分配给这个节点的地址
# public-key要填写上面在ECS上得到的publickey
# preshared-key相当于密码，用来保护这个peer不会被陌生的节点连接
# comment按需填写，用来标识这个节点
/interface/wireguard/peers add allowed-address=10.10.0.100/32 interface=wireguard1 public-key="ECS-PUBLIC-KEY" preshared-key="the-preshared-key" comment=ECS-SH-1
```

接下来到ECS，在`/etc/wireguard`目录下创建配置文件`wg0.conf`，填写如下内容：

```ini
[Interface]
PrivateKey = <填写上面生成的私钥>
Address = 10.10.0.100/32
DNS = 119.29.29.29

[Peer]
PublicKey = <填写RouterOS的公钥>
PresharedKey = <填写在RouterOS中Peer的preshared-key>
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 0
Endpoint = <你的公网IP或DDNS域名>:13231
```

保存后就可以启动`wg0`这个接口并测试组网是否成功。

```bash
# 启动wg0
$ sudo systemctl start wg-quick@wg0

# ping一个内网地址看有没有响应
$ ping 192.168.1.20
PING 192.168.1.20 (192.168.1.20) 56(84) bytes of data.
64 bytes from 192.168.1.20: icmp_seq=1 ttl=63 time=6.07 ms
64 bytes from 192.168.1.20: icmp_seq=2 ttl=63 time=4.72 ms
64 bytes from 192.168.1.20: icmp_seq=3 ttl=63 time=5.25 ms
64 bytes from 192.168.1.20: icmp_seq=4 ttl=63 time=6.07 ms
```

如果能从ECS上ping通LAN侧的主机，那就说明组网成功了。接下来可以让`wg0`接口在开机后自动启动：

```bash
sudo systemctl enable wg-quick@wg0
```

## 故障恢复

文章发布之后没两天，我的软路由突然内核崩溃重启，导致我的公网IPv4地址变了，而Wireguard在建立连接后，并不会检查对端的地址是否有变化，一旦变化，就会导致Wireguard隧道不通。于是我写了一段小脚本放在crontab里，每分钟ping一次对端的Wireguard接口地址，如果不通，就重启Wireguard服务。

```bash
#!/bin/bash

# Ping对端Wireguard地址，仅ping一次，超时1秒
ping 10.10.0.1 -qc 1 -w 1 > /dev/null
# 如果ping不成功，那么返回码是1
RESULT=$?

# 只要返回码不是0，就重启Wireguard连接
if [ $RESULT -ne 0 ]; then
  systemctl restart wg-quick@wg0
fi
```
