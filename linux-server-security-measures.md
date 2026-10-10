---
title: "购买新的Linux服务器后需要做的安全措施"
source: https://www.boris1993.com/linux-server-security-measures.html
date: 2018-09-02
updated: 2026-10-10
tags: [Linux]
categories: [瞎折腾]
---

# 购买新的Linux服务器后需要做的安全措施

在购买了一台安装Linux系统的VPS之后，首先要做好一定的安全防护措施，来尽可能地保证你的VPS不会沦为某些人的肉鸡。

# 修改SSH默认端口

这个世界上，总有不少无聊的人做着SSH弱口令扫描这种事情，相信我，你的IP一定是在被扫描的IP段里的，所以，更换掉SSH的默认端口会是个好主意。

用你喜欢的文本编辑器，编辑 `/etc/ssh/sshd_config` ，修改如下内容：

```plaintext
# SSH默认端口号为22
# 将其修改为一个自定义的端口号，比如15
Port 15
```

保存配置文件，并重启SSH服务端进程，使新的配置生效。**注意此时不要退出现在的SSH会话，因为一旦出现问题，我们还可以使用这个会话来修复，否则就只能通过虚拟控制台来登陆了**

接下来新开一个SSH会话，记得通过新指定的端口号来连接，以检查配置是否正确无误。没问题之后，就可以关掉其中一个SSH会话了，毕竟还是不要浪费资源，注意环保不是。

# 启用防火墙

一台放在公网上的，没有防火墙的服务器，那就是一台公交车。那么怎么变成私家车呢？废话，开防火墙啊！

目前已经有一些iptables的前端工具，来简化防火墙的配置了，所以不建议直接操作iptables。当然如果您有信心，那就当我没说。

不同发行版所使用的iptables前端可能是不一样的，比如CentOS使用的是 `firewalld` ，而Ubuntu使用的是 `ufw` 。由于我使用的是Ubuntu，所以下面就以 `ufw` 来举例了，使用 `firewalld` 的同学，可以参考这篇文章来学习。

```bash
# 首先检查防火墙是否在运行
$ sudo ufw status
Status: inactive # 说明目前ufw并没有启动

# 启动ufw
$ sudo ufw enable
# 防火墙已激活，并会跟随系统自动启动
Firewall is active and enabled on system startup

# 把SSH的端口加入到开放的列表中
# 否则这个会话断掉之后，你就再也别想用SSH登陆了
$ ufw allow 15
Rule added
Rule added (v6)

# 看一下是不是成功加进去了
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
15                         ALLOW       Anywhere
15 (v6)                    ALLOW       Anywhere (v6)
```

这样防火墙就启动了，并且仅接受来自15端口的入站请求。

在配置放行列表的时候，注意仅添加必要的端口。乱开一气，或者直接允许所有端口，那就跟没有防火墙一样了。

# 禁止root用户通过SSH登陆

地球人都知道，root是Linux系统中权限最高的用户，同时也是最危险的用户。所以当然不能开放root远程登录的权限。

在此之前，你需要先创建一个自己的用户，并且设置好密码，保证可以正常登陆到系统中。至于创建用户的方法我这里就不赘述了。

有了自己的用户之后，编辑 `/etc/ssh/sshd_config` ，修改如下内容：

```plaintext
# 允许使用root用户登陆
PermitRootLogin no
```

保存并重启SSH服务，就可以阻止root用户通过SSH登陆了。

# 使用RSA密钥对登陆SSH

在防止被破解的角度上，光是换端口和开防火墙，是远远不够的。还记得上文提到的SSH弱口令扫描吗？相信我，你绞尽脑汁想出来的密码，还真不一定打得过字典，而VPS提供商给你的默认密码，你确定你能记得住？但是RSA密钥对不仅能抵抗字典攻击，还不需要你记忆什么登陆信息，何乐而不为？

## 生成密钥对

### Windows下生成密钥对

如果你用的是Windows，那么可以参考这篇文章来生成你的密钥对，并将其上传至服务器。

### Linux下生成密钥对

如果你用的是Linux，那么可以使用 `ssh-keygen` 命令生成密钥对。

首先在本地电脑上生成密钥对

```text
$ ssh-keygen -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/boris/.ssh/id_rsa): 密钥文件的位置，回车使用默认值
Enter passphrase (empty for no passphrase): 密钥的密码，留空表示没有密码
Enter same passphrase again: 确认密码
Your identification has been saved in /Users/boris/.ssh/id_rsa.
Your public key has been saved in /Users/boris/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:K23EGsTSYVod8LbR/6MRHm3XHJbAlTxCbewMUXmfaOo boris@Boris-MacBook-Pro.local
The key's randomart image is:
+---[RSA 2048]----+
|      =o..  o+Boo|
|     * o..   +.Oo|
|    o + + .   B+=|
|     o o o . +.=+|
|      . S   * o +|
|       = . o = . |
|  .   o + . o o  |
|       o   E o . |
|   +        .    |
+----[SHA256]-----+
```

生成好了之后，就可以使用 `ssh-copy-id` 来传输公钥到远程服务器了：

```bash
ssh-copy-id -i ~/.ssh/id_rsa <USERNAME>@<HOST>
```

然后就可以使用密钥登陆来测试了，方法很简单，只需要为 `ssh` 命令添加 `-i` 参数并指定私钥文件即可，如果私钥文件名为 `id_rsa` 的话， `-i` 参数也可以省略了。

如果觉得使用密钥登陆的命令过长，那么可以参考我的另一篇博客[使用 SSH config 简化 SSH 连接](https://www.boris1993.com/tools/ssh/ssh-config-file.html)来简化SSH客户端的操作。

## 配置SSH仅接受密钥登陆

在服务器上编辑 `/etc/ssh/sshd_config` 修改如下内容：

```plaintext
# 启用公钥认证
PubkeyAuthentication yes
# 禁用密码认证
PasswordAuthentication no
# 不允许空密码登陆
PermitEmptyPasswords no
```

保存文件并重启SSH服务，然后服务器就只会接受通过密钥认证的登陆请求了，在密钥错误或者未提供密钥的时候，服务器会直接拒绝请求。

做完以上几步之后，通常的端口扫描和弱口令攻击基本上就免疫了，但是安全没有终点，在日常维护中还是需要时常检查鉴权日志和防火墙日志，以保证系统仍处于安全的状态下。在配置其他应用的时候，也要考虑到应用的安全相关的配置。
