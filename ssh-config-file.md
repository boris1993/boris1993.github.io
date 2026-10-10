---
title: "使用SSH config简化SSH连接"
source: https://www.boris1993.com/ssh-config-file.html
date: 2018-01-09
updated: 2026-10-10
tags: [Shell, SSH, ssh_config, ProxyCommand, ProxyJump, 跳板机]
categories: [小技巧]
---

# 使用SSH config简化SSH连接

如果你有很多的服务器要连接，如果对你来说记住那些服务器的地址、端口实在是一种痛苦，如果你一直在寻找一种能够简化在命令行下连接SSH服务器的办法，那么，本文将给你提供一种解决问题的思路，那就是，使用SSH的config文件。

# SSH config文件是什么

Open SSH客户端配置文件，允许你以配置项的形式，记录各个服务器的连接信息，并允许你使用一个定义好的别名来代替其对应的ssh命令参数。

# SSH config文件该怎么用

## 创建SSH config文件

通常来说，该文件会出现在两个地方，一个是`/etc/ssh/ssh_config`，一个是`~/.ssh/config`。

`/etc/ssh/ssh_config`文件通常用来定义全局范围上的SSH客户端参数，而`~/.ssh/config`则被用来定义每个用户自己的SSH客户端的配置。我们将要修改的，就是位于用户目录下的config文件。

如果`~/.ssh/config`文件不存在，那么也不用着急，这是正常的，只需要执行如下命令，即可新建一个空白的config文件

```bash
touch ~/.ssh/config
```

## 编写config条目

假如说，我们想连接到一台服务器，它的地址是example.server.com，端口号是2222，以用户admin登陆，并使用~/.ssh/id\_rsa这个私钥验证身份。那么，我们需要在命令行里输入：

```bash
ssh admin@example.server.com -p 2222 -i ~/.ssh/id_rsa
```

嗯好吧，-i参数可以省略，但即使这样，命令还是很长，对吧？

那么我们把这个服务器的连接参数写到config文件里，就变成了这个样子：

```text
# 此处我为了美观起见，给每个子条目都缩进了一层，实际使用时缩进不影响文件的效果。

Host sample
    Hostname        example.server.com
    Port            2222
    User            admin
    Identityfile    ~/.ssh/id_rsa
```

嗯，在这里，它还有了一个新名字，叫`sample`。

然后，我们只需要：

```bash
ssh sample
```

就可以连接到这台主机了。

## 创建通配符规则

有的时候，我们需要连接多台不同的主机，那难道我们需要针对每个主机都写一遍规则吗？

答案是不一定。如果你要连接的主机，它们的域名有一定规律可循，那么我们可以用通配符来匹配这一系列的主机。

比如，公司里针对开发和测试环境，各创建了一系列集群，同时各个集群中又根据负责的业务不同，有多台负载均衡的主机，那么我们可以这样写：

```text
# 首先匹配所有主机，在这里配置好我们的用户名、私钥等参数
Host *
    User            boris1993
    IdentityFile    ~/.ssh/id_rsa

# 开发环境
Host dev-*
    HostName    %h.dev.mycompany.com

# 测试环境
Host staging-*
    HostName    %h.staging.mycompany.com
```

接下来，我们只需要执行类似`ssh dev-user`，就可以连接到开发环境的负责用户管理的服务器上了。

而实现这个操作的重点，我想你已经注意到了，一个是`Host`配置中的`*`，另一个就是配置文件里面的`%h`。星号我们都知道，是个通配符，放在这里就意味着它会匹配所有以`dev-*`开头的主机名；而`%h`是一个占位符，它会把你在`SSH`命令中`destination`部分的输入取出来放在这里，所以当我们执行`ssh dev-user`的时候，实际上命令会被展开成`ssh dev-user.dev.mycompany.com`。

## 配置通过跳板机连接

如果公司规定所有服务器都需要先`SSH`到跳板机，然后才能连接到具体的服务器，那该怎么办？把`ssh_config`文件放在跳板机上吗？

确实这是一个解决方案，但不是唯一解。因为我们还有一个配置参数叫`ProxyJump`。它，就是用来指定，在连接这个服务器之前，需要通过哪个跳板机。

写起来，是这样的：

```text
# 其他部分略

# 跳板机
Host bastion
    HostName    bastion.mycompany.com

# 服务器
Host dev-*
    HostName    %h.dev.mycompany.com 
    ProxyJump   bastion
```

如果你的`SSH`版本早于7.3，那么很可惜它是不支持`ProxyJump`这条配置的。但是不要灰心，它支持另一条配置，`ProxyCommand`。

```text
# 其他部分略

# 跳板机
Host bastion
    HostName    bastion.mycompany.com

# 服务器
Host dev-*
    HostName        %h.dev.mycompany.com 
    ProxyCommand    ssh -W %h:%p bastion 
```

然后，我们继续执行`ssh dev-user`就可以了，`SSH`会自动先连接到跳板机，然后在跳板机中再连接到我们要去的服务器。

如果你们公司安全做的非常好，需要通过数个跳板机来连接服务器，那么只需要给每个跳板机都配置上更外层的跳板就好了，像这样：

```text
# 其他部分略

# 最外层跳板机
Host bastion-outer
    HostName    bastion-outer.mycompany.com

# 下一层跳板机
Host bastion-inner
    HostName    bastion-inner.mycompany.com
    ProxyJump   bastion-outer

# 服务器
Host dev-*
    HostName    %h.dev.mycompany.com 
    ProxyJump   bastion-inner
```

这样，`SSH`就会先连接到`bastion-outer`，然后连到`bastion-inner`，然后连到`dev-user`。理论上，你可以在这里无限套娃（逃

# 这玩意有意思，我还想了解更多！

好吧，为了满足你的好奇心，我这里为你提供了3篇博客供你参考。当然，这三篇博客也是我编写本文时的参考文档。

多个 SSH KEY 的管理

How To Configure Custom Connection Options for your SSH Client

Simplify Your Life With an SSH Config File

另外，您也可以阅读ssh\_config的手册页，来获得最原始的信息，阅读该手册的命令是：

```bash
man ssh_config
```
