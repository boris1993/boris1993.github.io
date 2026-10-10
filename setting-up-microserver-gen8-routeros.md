---
title: "二手MicroServer Gen8折腾记(2/n) - 配置RouterOS软路由"
source: https://www.boris1993.com/setting-up-microserver-gen8-routeros.html
date: 2023-02-14
updated: 2026-10-10
tags: [RouterOS, 二手MicroServer Gen8折腾记, Gen8, 软路由]
categories: [瞎折腾]
---

# 二手MicroServer Gen8折腾记(2/n) - 配置RouterOS软路由

我之前宽带一直都是光猫拨号，电脑和NAS直连光猫，光猫下挂个二级无线路由器来给其他设备联网，但我早就对这个拓扑不爽了，主要原因是自己花大钱买的AC86U派不上用场，只能干点边缘的杂活，而且搞两套局域网怎么看都挺恶心的。之前也试过把路由器改成AP模式，让所有设备都直连光猫，但是万恶的电信光猫限制了设备数，只有前5台机器能上网，其它的机器虽然能拿到地址，但是上网必定超时。那现在既然有了台服务器，那干脆装个软路由。

软路由系统我打算玩玩`RouterOS`，毕竟`iKuai`听说过没用过，据说还出过劫持流量这种问题，不敢用也不想用；`OpenWRT`以前玩过，没啥新鲜感，而`RouterOS`，之前装着玩的时候就觉得倍有逼格，早就想搞搞它玩玩了。

本篇为[二手MicroServer Gen8折腾记](https://www.boris1993.com/tags/%E4%BA%8C%E6%89%8BMicroServer-Gen8%E6%8A%98%E8%85%BE%E8%AE%B0/)的第二篇，记录配置猫棒和搭建RouterOS软路由的过程。

## 获取光猫中的配置

我找装维师傅说要玩猫棒，需要进后台看参数，师傅就把密码查给我了。

用超级密码登录光猫后台，把LOID、PPPoE的用户名和密码（密码需要base64解码才能得到真实的密码）、IPv6前缀长度记在一个文档里备查。

## 准备猫棒

因为我懒得找运营商的装维师傅给我改桥接（之前沟通过，师傅说要先换光猫，新光猫又得等），也考虑到留着原来的光猫配置不变的话，在我折腾翻车的时候也不需要再找装维师傅改回路由模式，直接把光纤接回去就能恢复网络，所以我直接在淘宝上买了根ODI的猫棒。又因为服务器上没有SFP口，我也不想再买个网卡，就顺便一起买了个光电转换器。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/converter-picture-on-taobao.png)

之前了解到，光纤的端面接口有PC、UPC、APC三种，之所以选择ODI的光猫，就是因为我们一般宽带光纤的端面是UPC的，而诺基亚和阿尔卡特之类的猫棒是APC接口，还要套个转换器才行，硬插进去可能会损坏光纤，但ODI这款采用的就是UPC接口，不用再买转换器。**但是我无法保证覆盖所有情况，如果你也想用猫棒，请一定提前跟店家和装维师傅确认好，如果顶坏了光纤端面，你只能喊装维师傅来给你重新做接头，别无他法！**

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/types-of-fiber-connector.jpg)

抓住光纤的蓝色滑套，往后抽，就可以把光纤从光猫上拔出来，然后对准接口插进猫棒，再把猫棒插进光电转换器，然后接上网线和电源，就可以了。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/ponstick-and-converter.jpg)

稍等几分钟等猫棒开机，开机之后，转换器在猫棒方向的灯会亮起来。这时候先把网线跟电脑连起来，给电脑分配一个`192.168.1.0/24`范围的地址（当然别用`192.168.1.1`，那是猫棒后台的地址），然后用浏览器打开`192.168.1.1`，就能看见猫棒后台的登陆页面了。账号和密码找商家要，我这款是`admin/admin`，但不保证所有ODI猫棒都是这一对账号密码。登陆进去后，进入`Settings`页面，填入`LOID`，点`Apply Changes`，然后进入`Commit/Reboot`页面，点`Commit and Reboot`按钮保存配置并重启，猫棒端就配置完成了。

不过电信、联通、移动的光猫注册方式各有不同，上面说的操作都是针对电信的。其他运营商可以参考猫棒商家发的文档，或参考网上其他人的成功案例。这里引用猫棒说明中的一部分内容：

> ★注册-初级:（部分型号光猫没有 GPON SN，可不修改）  
> 电信&联通 GPON/XGPON/XGSPON：Loid  
> 移动 GPON/XGPON/XGSPON：Ploam 密码  
> 电信&联通&移动 EPON/10G EPON：Loid
> 
> ★注册-中级:  
> 电信&联&移动 GPON/XGPON/XGSPON：GPON SN  
> 电信&联通&移动 EPON/10G EPON：MAC/PON MAC+MAC KEY
> 
> ★注册-高级：（GPON/XGPON/XGSPON）  
> 厂商 ID/Vendor ID：如 HWTC/华为、FHTT/烽火、ZTEG/中兴、SCTY/天邑、YHTC/友华  
> 设备型号  
> OUI：6 位  
> 设备标号：一般为 17 位  
> 软件版本号  
> ONT 版本/硬件版本号
> 
> ★注册-高级：（EPON/10G EPON）  
> 设备型号  
> ONT 版本/硬件版本号

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/pon-stick-settings-loid.png)

本来重启之后，应该先看一下ONU状态是不是`O5`来着，但是当时我忘了看，直接电脑上拨号成功，间接也证明了LOID注册成功。不过还是放个文档里扒出来的图好了。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/ponstick-status-o5.png)

## 安装RouterOS

一开始我以为装这玩意跟装别的系统没啥区别，就ISO一挂，顺着向导装就行。可看了文档发现，RouterOS有一个专门面向虚拟平台的版本，叫`Cloud Hosted Router (CHR)`，在官网的下载页面下载`OVA Template`，然后导入ESXi就行。具体的安装方法可以看Mikrotik的Wiki。

`RouterOS CHR`作为一个付费的系统，当然也需要买授权才能用。买授权的方法也可以跟着Mikrotik的Wiki操作，就是注意别买成`RouterOS`的Key，这两者是不通用的。

另外安装之后，如果你更改了RouterOS的CPU数量，那么内存大小也要跟着调整。官方文档提供了如下公式：

> RouterOS 6：RAM = 128 + \[8 \* CPU数量 \* (网络接口数量 - 1)\]  
> RouterOS 7：RAM = 256 + \[8 \* CPU数量 \* (网络接口数量 - 1)\]  
> 注：建议为CHR实例分配至少1024MB内存。

我因为没注意到这个信息，只把CPU改成了4核心，但内存还是160MB，导致几乎每24小时就会出一次`kernel failure in previous boot`错误，为这事我还给MikroTik开了个工单……

## 配置ESXi的网络

如果我没记错的话，ESXi一开始只给Ethernet 1口做了配置，我们得手动给Ethernet 2配置虚拟网络。

首先到`网络`的`虚拟交换机`页面，创建一个新的虚拟交换机，名字按自己喜好起就行，比如我就顺着已有的`vSwitch0`给它命名`vSwitch1`。然后点进`vSwitch1`，点`添加上行链路`，把`vmnic1`分配给它，这样在ESXi里面就可以用这个接口了。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/esxi-vswitch1.png)

然后到`端口组`页面，添加一个新的端口组`VM Network 2`，虚拟交换机选择刚创建的`vSwitch1`。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/vm-network-2.png)

接下来点进虚拟机`RouterOS`，关机之后编辑设置，给它添加一个网络适配器，然后给`网络适配器1`分配`VM Network 2`，给`网络适配器2`分配`VM Network 1`，保存，开机。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/vm-routeros-settings.png)

## 配置RouterOS

如果猫棒的ONU状态是O5，那就可以把转换器连到Gen 8的Ethernet 2口上了。你问为啥不是Ethernet 1？因为Ethernet 1我改成了与iLO共享，所以只能拿Ethernet 2当WAN口。下面进入RouterOS，开始配置软路由。

### 首次登录

第一次登录的时候，因为我并不知道路由器的地址，所以没办法用IP地址打开RouterOS的配置页面，但MikroTik很贴心的提供了用MAC地址连接的功能。在WinBox的Neighbors页面双击RouterOS的条目，用`admin`用户和空密码登录就可以。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/routeros-login-with-mac-address.png)

为了后面分的清楚，我先到`Interfaces`页面，把LAN口和WAN口对应接口的名字改了过来，毕竟默认的`ether1`和`ether2`不如`LAN`和`WAN`看的清楚。通常来说观察接口的`Tx`和`Rx`就能分清哪个是LAN口，因为这时候没配置拨号，WAN口的流量通常只会是0。如果分不清，那就先把Gen 8的Ethernet 1拔了。

![请暂时忽略China Telecom和wireguard1](https://blog-static.boris1993.com/setting-up-microserver-gen8/routeros-interfaces.png)

### 规划地址范围，配置DHCP、DNS

在配置DHCP服务器前，要先规划好自己的局域网的网段。地址范围可以从IPv4的三个私有地址范围`10.0.0.0/8`、`172.16.0.0/12`和`192.168.0.0/16`之间选，一定不要作死用公网网段（比如一些无能IT给路由器配了个`1.1.1.1`），不然我很难说你的网络会出什么奇怪的问题。我干脆就遵循之前的网络，用了`192.168.1.0/24`段作为我的内网网段。

确定好地址范围之后，就可以用这条命令给LAN口指定一个静态IP地址：

```routeros
/ip/address/add \
  interface=LAN \
  address=192.168.1.3/24 \
  network=192.168.1.0
```

选择给RouterOS分配`192.168.1.3`的原因是，`192.168.1.1`留给猫棒（虽然后来发现连不上），`192.168.1.2`留给AC86U路由器，它本来就是这个地址，我懒得再动它。

然后先给DHCP服务器添加一个地址池：

```routeros
/ip/pool/add \
  name=192.168.1.0/24 \
  ranges=192.168.1.2-192.168.1.254
```

接下来配置DHCP服务器：

```routeros
# 添加DHCP服务器
/ip/dhcp-server/add \
  name=LAN \
  address-pool=192.168.1.0/24 \
  interface=LAN \
  lease-time=1d

# 配置DHCP服务器下发的网络参数
/ip/dhcp-server/network/add \
  address=192.168.1.0/24 \
  dns-server=192.168.1.3 \
  gateway=192.168.1.3
```

此外我还想要给ESXi之类的机器配置一个固定的地址，于是：

```routeros
/ip/dhcp-server/lease/add \
  address=192.168.1.4 \
  mac-address=D0:BF:9C:46:94:D8 \
  server=LAN \
  comment=ESXi
```

上面DHCP网络参数中的`dns-server=192.168.1.3`配置的是让RouterOS作为DNS服务器，但现在RouterOS还不能响应远程的DNS请求，所以要用下面这条命令更改DNS的配置：

```routeros
/ip/dns/set allow-remote-requests=yes
```

### 创建本地管理员，禁用admin

安全起见，我建议给RouterOS创建一个新的管理员账户，并禁用默认的`admin`账户。

```routeros
# 创建新的管理员用户，仅允许在192.168.1.0/24网络内登录
/user/add \
  name=新用户的用户名 \
  password=新用户的密码 \
  group=full \
  address=192.168.1.0/24
# 禁用admin
/user/set [ find name="admin" ] disabled=yes
```

### 配置PPPoE拨号和NAT规则

首先创建一个PPPoE客户端来拨号：

```routeros
/interface/pppoe-client/add \
  name="PPPoE客户端名字" \
  interface=WAN \
  user="宽带账号" \
  password="宽带密码" \
  profile=default \
  use-peer-dns=yes \
  add-default-route=yes
```

添加成功后，RouterOS会自动开始拨号，在终端也可以用`monitor`命令实时监控PPPoE拨号端口的状态：

```routeros
/interface/pppoe-client/monitor [ find running ]
               status: connected
               uptime: 13h59m53s
         active-links: 1
             encoding: 
         service-name: 
              ac-name: SH-SH-ML-MSE-2.MAN.NE40E
               ac-mac: D0:D0:4B:06:EC:CC
                  mtu: 1492
                  mru: 1492
        local-address: 124.78.123.123
       remote-address: 124.78.123.1
   local-ipv6-address: fe80::5
  remote-ipv6-address: fe80::d2d0:4bff:fe06:eccc
-- [Q quit|D dump|C-z pause]
```

这时候RouterOS自己已经可以上网了，但是路由器下面的设备还不行，因为没有配置NAT。在终端执行这条命令，来创建一条NAT规则：

```routeros
/ip/firewall/nat/add \
  chain=srcnat \
  action=masquerade \
  out-interface="PPPoE接口的名字" \
  comment=NAT
```

现在路由下的设备就可以上网了。

### 配置IPv6

电信宽带已经支持了IPv6，那既然有了，我不能不用。电信会给宽带分配一个前缀，下级设备通过前缀来分配自己的地址，就是prefix delegation。所以我只需要在RouterOS中配置一个IPv6的DHCP client就行。

```routeros
/ipv6/dhcp-client/add \
  interface="PPPoE客户端名字" \
  add-default-route=yes \
  pool-name=ipv6_pool \
  pool-prefix-length=60 \
  prefix-hint=::/56 \
  request=prefix \
  use-interface-duid=yes
```

然后检查DHCP client的状态，是`bound`的话就说明已经成功获取到地址和前缀。

```routeros
/ipv6/dhcp-client/print
Columns: INTERFACE, STATUS, REQUEST, PREFIX
# INTERFACE      STATUS  REQUEST  PREFIX                              
0 China Telecom  bound   prefix   240e:38a:5d27:5c00::/56, 2d21h48m59s
```

然后让电脑重新连接路由，就能看到电脑成功获取到IPv6地址了。

### 配置防火墙

毕竟路由器这种玩意不能在网上裸奔，该禁的端口就得禁。

首先创建两个interface list，把公网接口和内网接口归类到各自的列表中，方便后面创建防火墙规则。

```routeros
/interface list
# 给内网接口的list
add name=local
# 给公网接口的list
add name=internet

/interface list member
# LAN口归进local这个list
add interface=LAN list=local
# WAN口和PPPoE口归进internet这个list
add interface=WAN list=internet
add interface="China Telecom" list=internet
```

然后添加IPv4的防火墙规则：

```routeros
/ip firewall filter
# 放行状态为established、related、untracked的数据包
add action=accept chain=input comment="accept established,related,untracked" connection-state=established,related,untracked
# 允许外网ping，不想允许的话就不执行这条
add action=accept chain=input comment="accept ping" protocol=icmp
# 丢弃invalid状态的包
add action=drop chain=input comment="drop invalid" connection-state=invalid
# 兜底规则，丢弃一切来自公网的包
add action=drop chain=input comment="drop all from WAN" in-interface-list=internet
# RouterOS的快速跟踪功能，可以跟踪已建立的连接并让这些流量走快速通道，以降低设备的CPU负载
# 好像必须要专门的硬件才能发挥作用，但总之先加在这里
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related hw-offload=yes
add action=accept chain=forward comment="accept established,related, untracked" connection-state=established,related,untracked
# 在forward链上也丢弃掉invalid状态的包
add action=drop chain=forward comment="drop invalid" connection-state=invalid
# 丢弃所有来自公网，且无端口转发规则匹配的包
add action=drop chain=forward comment="drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=internet
```

对于IPv6，要先创建两份address list，分别包含可以放行的地址和有问题的地址：

```routeros
/ipv6 firewall address-list
add address=fe80::/16 list=allowed
add address=ff02::/16 comment=multicast list=allowed
add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
add address=::1/128 comment="defconf: lo" list=bad_ipv6
add address=fec0::/10 comment="defconf: site-local" list=bad_ipv6
add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" list=bad_ipv6
add address=::/96 comment="defconf: ipv4 compat" list=bad_ipv6
add address=100::/64 comment="defconf: discard only " list=bad_ipv6
add address=2001:db8::/32 comment="defconf: documentation" list=bad_ipv6
add address=2001:10::/28 comment="defconf: ORCHID" list=bad_ipv6
add address=3ffe::/16 comment="defconf: 6bone" list=bad_ipv6
add address=::224.0.0.0/100 comment="defconf: other" list=bad_ipv6
add address=::127.0.0.0/104 comment="defconf: other" list=bad_ipv6
add address=::/104 comment="defconf: other" list=bad_ipv6
add address=::255.0.0.0/104 comment="defconf: other" list=bad_ipv6
```

然后配置防火墙规则（其中一些规则我也不明白是啥作用，但是既然包含在默认配置里，那配进去应该不会造成什么问题）：

```routeros
/ipv6 firewall filter
# 放行状态为established、related、untracked的数据包
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
# 丢弃invalid状态的包
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
# 允许IPv6 ping
add action=accept chain=input comment="defconf: accept ICMPv6" protocol=icmpv6
# 允许IPv6 traceroute
add action=accept chain=input comment="defconf: accept UDP traceroute" port=33434-33534 protocol=udp
# 允许DHCPv6 prefix delegation的包
add action=accept chain=input comment="defconf: accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address=fe80::/16
# 允许IKE数据包
add action=accept chain=input comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=input comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=input comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=input comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
# 兜底规则，丢弃一切不是从local接口列表发出的包
add action=drop chain=input comment="defconf: drop everything else not coming from LAN" in-interface-list=!local
# forward链上放行状态为established、related、untracked的数据包
add action=accept chain=forward comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
# forward链上丢弃invalid状态的包
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
# 禁止来自bad_ipv6这个地址列表里面的地址
add action=drop chain=forward comment="defconf: drop packets with bad src ipv6" src-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: drop packets with bad dst ipv6" dst-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: rfc4890 drop hop-limit=1" hop-limit=equal:1 protocol=icmpv6
add action=accept chain=forward comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=forward comment="defconf: accept HIP" protocol=139
add action=accept chain=forward comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=forward comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=forward comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=forward comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
add action=drop chain=forward comment="defconf: drop everything else not coming from LAN" in-interface-list=!local
```

## RouterOS里一些其它的配置

最后就是一些杂七杂八的配置，比如开启NTP客户端来自动同步时间：

```routeros
/system ntp client
set enabled=yes
/system ntp client servers
add address=time.windows.com
add address=time.nist.gov
add address=time.apple.com
```

禁用掉不需要的服务以减小攻击面：

```routeros
/ip service
# 禁用telnet
set telnet disabled=yes
# 禁用ftp
set ftp disabled=yes
# 限制www、ssh、winbox服务都只能从内网访问
set www address=192.168.1.0/24
set ssh address=192.168.1.0/24
set www-ssl address=192.168.1.0/24 disabled=no
set winbox address=192.168.1.0/24
# API用不到，所以禁用
set api disabled=yes
set api-ssl disabled=yes
```

开启UPnP，虽然说开这玩意会有安全隐患，但是开了这么多年也没因为它出过事，这玩意也没法从外网开端口转发，也就是说首先内网得有机器中毒，我自认为我的使用习惯还算是安全的，所以，开。

```routeros
# 启用UPnP
/ip upnp
set enabled=yes

/ip upnp interfaces
# 配置LAN口作为内网端
add interface=LAN type=internal
# 配置PPPoE端口作为公网端
add interface="PPPoE接口的名字" type=external
```

开启云服务，包括DDNS和自动备份：

```routeros
/ip cloud
set ddns-enabled=yes ddns-update-interval=1h update-time=yes
```
