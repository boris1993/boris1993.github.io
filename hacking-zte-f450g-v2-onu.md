---
title: "为了改桥接，我决定破解中兴F450G V2光猫"
source: https://www.boris1993.com/hacking-zte-f450g-v2-onu.html
date: 2023-03-16
updated: 2026-10-10
tags: [光猫, 桥接, F450G]
categories: [瞎折腾]
---

# 为了改桥接，我决定破解中兴F450G V2光猫

还记得我[之前买了个猫棒来替换光猫](https://www.boris1993.com/setting-up-microserver-gen8-routeros.html)么？用了一个来月，发现这玩意真的不稳定，短则几分钟长则一两天它必定自己重启一次，导致我的网络时不时就会断线。这玩意不好使，我也没有别的光猫，只好找电信装维师傅给我改桥接。可哪成想一个师傅踢皮球让我找另一个师傅，另一个一听我说是`悦me`直接就说不能改，只有SDN光猫能改。那你们逼我的，我自己改。

## 光猫信息

我的这款光猫是`EPON上行悦me网关`，型号是`中兴 ZXHN F450G`，区域是上海，硬件版本`V2.0`。

![](https://blog-static.boris1993.com/hacking-zte-f450g-v2-onu/f450g-onu-info.png)

## 准备工作

开始折腾光猫之前，一定要把当前光猫里存的配置备份好，尤其是`LOID`，万一折腾砸了，还可以改回去，或者重新注册，让局端重新下发正确的配置。我已经有了`telecomadmin`的密码，就直接登进后台，把配置抄了下来。

![](https://blog-static.boris1993.com/hacking-zte-f450g-v2-onu/onu-settings-backup.png)

## 开始动手

> 首先我要大力感谢这篇帖子，那天我就是照着这里面的内容操作的。  
> https://www.chinadsl.net/thread-127666-1-1.html

### 复位光猫，开启telnet

首先，拔掉光纤，用细针捅住光猫后面的复位孔，直到光猫前面板下方的两个灯都熄灭，只剩下上面的电信logo亮着，以彻底复位光猫。这一步是为了重新打开telnet，因为这个光猫在插上光纤之后telnet就会被禁用，只有拔掉光纤并彻底复位才能再把telnet打开。

复位后光猫会自动重启，拿根网线，把电脑接上光猫的LAN口（别接LAN 2，那是给IPTV用的），电脑设定静态IP地址`192.168.1.100`（其实除了`192.168.1.1`之外，整个`192.168.1.0/24`网段随便你选），子网掩码`255.255.255.0`。

接下来持续`ping 192.168.1.1`，光猫启动好之后会有响应，接下来用telnet连接192.168.1.1，使用用户名`root`和密码`Zte521`登进光猫shell。

### 需要通过telnet做的配置

首先用以下命令实现在光纤插入后telnet仍然可用。

```bash
# 打开telnet
sendcmd 1 DB set TelnetCfg 0 TS_Enable 1
# 允许局域网登陆
sendcmd 1 DB set TelnetCfg 0 Lan_Enable 1
# 插上光纤后不关闭telnet
sendcmd 1 DB set TelnetCfg 0 Lan_EnableAfterOlt 1
# 最多允许5个用户同时登陆telnet，避免自己被锁在外面
sendcmd 1 DB set TelnetCfg 0 Max_Con_Num 5
# 打开23端口
sendcmd 1 DB set PortControl 3 PortEnable 1
# 保存配置
sendcmd 1 DB save
```

然后可以用`sendcmd 1 DB p TelnetCfg`确认下配置。后面的操作也可以用`sendcmd 1 DB p 表名`这条命令确认配置，就不再重复了。

```xml
<Tbl name="TelnetCfg" RowCount="1">
        <Row No="0">
                <DM name="TS_Enable" val="1"/>
                <DM name="Wan_Enable" val="0"/>
                <DM name="Lan_Enable" val="1"/>
                <DM name="TS_Port" val="23"/>
                <DM name="TSLan_Port" val="23"/>
                <DM name="TS_UName" val="root"/>
                <DM name="TS_UPwd" val="Zte521"/>
                <DM name="TSLan_UName" val="root"/>
                <DM name="TSLan_UPwd" val="Zte521"/>
                <DM name="Max_Con_Num" val="5"/>
                <DM name="ProcType" val="0"/>
                <DM name="Lan_EnableAfterOlt" val="1"/>
                <DM name="TS_Sprtwl_Mode" val="0"/>
                <DM name="WanWebLinkToTS" val="0"/>
        </Row>
</Tbl>
```

然后禁用掉TR069。电信局端通过TR069协议下发配置，而我们自己改的桥接肯定跟局端的配置不一样，不关掉这玩意相当于白忙活。

```bash
# 禁用TR069的58000端口
sendcmd 1 DB set PortControl 6 PortEnable 0
sendcmd 1 DB save
```

虽然原帖中说还要把远程管理配置乱改以斩草除根，我是懒得改，应该关了端口就行，因为一来直到现在一个星期了我的配置都还在，二来重新设备注册的时候也会卡在`等待ITMS下发配置`，所以TR069应该是已经被掐了。

然后需要骗过光猫，让它认为ITMS下发配置已经成功。一开始搞的时候没做这一步，就发现过几分钟网页就打不开，nslookup一下发现解析全去了192.168.1.1，又打开个网站说试试，结果直接被光猫劫持到了设备注册页面。直接在光猫的表里面改掉ITMS状态，再清理DNS缓存之后就正常了。

```bash
sendcmd 1 DB set PDTCTUSERINFO 0 Status 0
sendcmd 1 DB set PDTCTUSERINFO 0 Result 1
sendcmd 1 DB save
```

接下来改掉光猫的最大用户数。其实要改桥接的话，这个改不改都没关系，反正我也不用光猫上网。但还是顺手改了。也不需要改什么最大用户数，直接关掉限制完事。

```bash
sendcmd 1 DB set CltLmt 8 Enable 0
sendcmd 1 DB save
```

至此telnet里面要搞的东西就都搞完了。虽然原帖里面还提到改掉`telecomadmin`的密码，但是既然TR069都废了，密码自然也不会再变。正好之前从装维师傅那要到的密码还能用，我也懒得再改。

### 登进后台配桥接

插上光纤，用telecomadmin登进光猫后台。先看一眼`网关注册信息`，因为光猫复位不会丢LOID，所以插上光纤之后，`光路（OLT）认证`应该就是认证成功的状态；而上面在telnet里面也改掉了ITMS认证状态，所以`管理（ITMS）注册`也应该是注册成功的状态。

![](https://blog-static.boris1993.com/hacking-zte-f450g-v2-onu/gateway-registration-status.png)

进入`网络`的`网络设置`，点开`连接名称`下拉菜单，看有没有`INTERNET_B_VID`，如果有，而且连接模式是`路由`，那就删了它。然后下拉菜单选`新建WAN连接`，连接模式选`桥接`，业务类型选`上网`，IP模式选`IPv4&IPv6`，点保存，光猫端就配置完成了。

![](https://blog-static.boris1993.com/hacking-zte-f450g-v2-onu/onu-wan-config.png)

## 齐活

到这光猫就改好桥接了，从光猫上扯根网线连上自己的路由器拨号，就可以上网了。正好改完桥接那天，RouterOS有更新，看PPPoE接口的uptime可以看出来，从RouterOS重启之后到现在，链接就没断过。之前用猫棒的时候，uptime就没多于过2天。

![](https://blog-static.boris1993.com/hacking-zte-f450g-v2-onu/routeros-connection-uptime.png)
