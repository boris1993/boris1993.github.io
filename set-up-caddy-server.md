---
title: "配置Caddy作为静态网站服务器和前置代理"
source: https://www.boris1993.com/set-up-caddy-server.html
date: 2018-07-19
updated: 2026-10-10
tags: [Caddy, WebServer]
categories: [瞎折腾]
---

# 配置Caddy作为静态网站服务器和前置代理

之前听闻有个新的Web Server，名曰Caddy，其配置简单，还默认启用HTTP/2，并且可以自动申请Let’s Encrypt的HTTPS证书。试用了一番，觉得不错，便把这个博客的服务程序换成了Caddy。在这里呢，记录一下安装和配置的过程。

# 安装

万事第一步，先安装。

## 下载页面概览

打开Caddy的下载页面，页面的内容简洁明了，左侧是4个要配置的项，右侧是每个配置项实际的内容。

![Download Page Overview](https://blog-static.boris1993.com/caddy/caddy-download-overview.png)

## 选择运行平台

首先，选择好Caddy要在哪个操作系统下运行。Caddy支持的平台还是足够多的，而且覆盖到了主流的操作系统，所以甭管您是Windows，还是Linux，抑或是BSD，都可以运行Caddy。因为我的服务器运行的是64位Ubuntu，所以选择`Linux 64-bit`。

实话说，看到Plan 9的时候，心里还是被惊到了。

![Choosing Platform](https://blog-static.boris1993.com/caddy/caddy-download-choose-platform.png)

## 选择插件

接下来是选择要安装哪些插件，通常来说，根据自己的需要来选择就可以了。如果后期要安装更多的插件的话，重新来下载页面勾选需要的插件并重新安装就可以了。毕竟是用Go写的，最后就一个可执行文件，替换掉原来的，就算重装好了。

我的需求有这么几点：

1.  我需要Caddy可以作为一个反向代理，所以选择了`http.forwardproxy`插件
2.  我的博客的源文件放置于我的GitHub中，我希望Caddy可以直接clone这个仓库，并且能通过WebHook监听这个仓库的更新事件，所以选择了`http.git`插件
3.  我在使用Cloudflare的DNS服务，并且Caddy可以通过DNS验证的方式申请HTTPS证书，所以需要`tls.dns.cloudflare`插件
4.  我想要Caddy作为一个系统服务，并且可以随系统自动启动，但是我又懒得自己写配置文件，所以使用`hook.service`插件来为我提供已经写好并经过了测试的配置文件

![Choosing Plugins](https://blog-static.boris1993.com/caddy/caddy-download-choose-plugins.png)

## 选择是否开启遥测功能

Caddy提供了一个叫做“遥测”的功能，可以监控您的Caddy实例的状态。具体针对该功能的描述，可以到其文档页面Telemetry阅读。这个功能开启与否与功能无关，开不开看您心情。

## 选择适合您的许可证

接下来，就是选择您要使用哪一种许可证来运行Caddy。像在下的博客是个人项目，不涉及商业应用，所以当然选择个人许可证。

![Choosing License](https://blog-static.boris1993.com/caddy/caddy-download-choose-license.png)

## 下载

终于，到了下载这一步了。Caddy提供了多种下载的方式，您可以在浏览器中将可执行文件下载到本地，或者通过命令行来下载，还可以直接使用一句话脚本来安装。

![Install Methods](https://blog-static.boris1993.com/caddy/caddy-download-methods.png)

如果使用自动安装脚本的话，Caddy会被安装到`/usr/local/bin/caddy`中。如果选择手动安装，那么需要将Caddy的可执行文件放到`PATH`所包含的目录中，或者将Caddy所在的位置加入到`PATH`中。

## 检查安装是否成功

完成安装后，可以直接使用`caddy`命令启动一个Caddy服务器，它会开始监听本机的`2015`端口，并列出当前工作目录的内容。使用`http://localhost:2015`即可访问。如果能成功打开，或者可以看到一个`404`页面，那么说明Caddy安装成功了。

# 配置

Caddy的所有配置都将被写到一个名为`Caddyfile`的文件中。点击这里阅读Caddy官方提供的入门指导，以及Caddy官方文档。

在以下实例中，我们假定`Caddyfile`的位置是`/etc/caddy/Caddyfile`，并且所有与Caddy相关的文件、目录，都存放于`/etc/caddy`下。

## 配置网站的地址

首先要配置Caddy所服务的网站的地址，如果只有一个地址的话，那么可以将地址写到`Caddyfile`的第一行，同时`Caddyfile`的第一行也必须是网站的地址。比如下面这样：

```plaintext
www.boris1993.tk
```

这样Caddy就会监听`www.boris1993.tk`所绑定的地址，并监听80端口提供HTTP服务，以及443端口提供HTTPS服务。在默认情况下，Caddy会自动将HTTP请求使用HTTP 301返回码重定向到HTTPS，除非显式配置禁用HTTPS服务。

如果需要指定端口号，那么可以在地址后面跟上端口号，比如`www.boris1993.tk:8080`。因为我没有用到这项功能，所以没有测试过这样配置的效果。如果您有需要还请自行测试。

如果要同时开启多个网站，那么各个网站的配置需要以大括号包围起来，比如下面这样：

```plaintext
www.boris1993.tk {
    
}

www2.boris1993.tk {

}
```

我们这里就只演示仅有一个地址的情况。多个地址的配置与单个地址的配置方式相同，故不再赘述。

## 配置静态文件所在的位置并启用gzip压缩

有了地址之后，我们需要告诉Caddy要提供的静态文件在什么位置，这个可以使用`root`指令来制定，如下面这样：

```plaintext
www.boris1993.tk {
    root    /var/www
}
```

然后Caddy就会到`/var/www`目录寻找`index.html`等默认的主页文件。

启用gzip压缩，可以使我们的网站打开的更快。在Caddy中启用gzip，也只需要一条指令：

```plaintext
www.boris1993.tk {
    root    /var/www
    gzip
}
```

## 提供申请HTTPS证书的信息

在默认情况下，Caddy会自动搞定申请HTTPS证书的事情，不需要用户进行干预。如果需要覆盖默认的配置，可以参考Caddy文档的TLS部分。

## 配置日志

### 访问日志

网站的访问日志可以使用`log`指令来配置，该指令的文档可以参考这里。

在这里我先放出我的配置，然后再逐行来解释。简明起见，我就只写出日志的部分，其余无关内容就不在这里写出来了。

```plaintext
log /   /var/log/caddy/access.log   "{combined}" {
    rotate_size 1
    rotate_age  7
    rotate_keep 2
    rotate_compress
}
```

第一行中，我指定要记录所有对网站根目录`/`的访问，将日志写到`/var/log/caddy/access.log`中，记录的方式是`combined`。

Caddy提供了两种日志格式，`common`和`combined`，`common`是默认的记录格式。

`common`的格式是这样子的：

`{客户端IP地址} - {HTTP基础验证的用户名} [{访问时间}] \"{HTTP方式} {请求的URI} {协议版本}\" {HTTP状态码} {响应体的大小}`

而`combined`格式，则是在`common`格式的末尾，追加如下内容：

`\"{>Referer}\" \"{>User-Agent}\"`

第二行`rotate_size`指定了在日志到达1MB大小之前不进行日志翻转，这个指令的单位是`MB`。

第三行`rotate_age`指定了保留7天的翻转日志。

第四行`rotate_keep`指定了只保留最近2个翻转日志，之前的版本将被删除。

第五行`rotate_compress`指定使用gzip压缩翻转日志。

### 错误日志

错误日志可以使用`errors`指令来配置，该指令的文档可以参考这里。

同样，我将以我的配置作为范例来解释，如果需要其他的配置可以参考官方文档。

```plaintext
errors  /var/log/Caddy/error.log {
    404         /var/www/error/HTTP404.html
    rotate_age  7
    rotate_keep 2
    rotate_compress
}
```

第一行配置了错误日志将被写入到`/var/log/Caddy/error.log`中。

第二行配置了当发生`404`错误后显示的页面，这里还可以为其他错误码指定错误页面，语法参见官方文档。

其余三行的含义与上文`log`指令中对应参数的含义一致，不再赘述。

## 配置自动从Git拉取页面内容

Caddy支持从一个指定的Git仓库克隆以及更新页面的内容到某个目录，并可以通过WebHook来监视仓库的更新，参考配置如下：

```plaintext
git https://github.com/boris1993/boris1993.github.io.git {
    path        /var/www
    hook        /hook       hook.password
    hook_type   github
}
```

这里我配置Caddy从`https://github.com/boris1993/boris1993.github.io.git`这个仓库拉取静态页面文件，这就是本博客所在的GitHub仓库，拉去之后文件将被放到`/var/www`目录下。因为我要实现博客文件自动更新，所以这里的地址需要与`root`指令配置的位置相同。

`hook`参数配置Caddy使用`www.boris1993.tk/hook`作为WebHook的监听地址，这个hook的访问密码是`hook.password`，并且使用下一行中的`hook_type`指令显式指定Hook的类型是`github`，也就是来自GitHub的hook。

这样配置完毕后，还需要为远程Git仓库配置hook，然后才可以实现自动更新。具体配置方式请参考Git仓库服务商的文档。

## 配置Caddy作为前置代理

一部分代理工具，比如v2ray，支持使用一个HTTP服务器作为其前置代理，Caddy就可以实现这样的功能。本示例中我配置Caddy作为v2ray的WebSocket代理，配置文件片段如下：

```plaintext
proxy /v2ray localhost:12345 {
    websocket
    header_upstream -Origin
}
```

这段配置指定了将`/v2ray`这个路径作为`localhost:12345`这个地址的前置代理，代理协议为`websocket`。具体的配置方法请参考被代理程序的文档。

## 配置开机自启动

`hook.service`插件可以一键生成`systemd`格式的自启动配置文件，只需要如下命令即可完成配置：

```bash
caddy -service install -conf /etc/caddy/Caddyfile
```

注意将`-conf`参数的值指向实际的`Caddyfile`的路径。

# 结束

至此，一个可以正常提供服务的Caddy服务器就配置完成了，现在Caddy可以提供正常的HTTP和HTTPS访问，并且会自动申请和续订HTTPS证书，在远端Git仓库有更新之后，Caddy也会自动更新本地的文件，一切都变成了自动化操作，正常情况下完全可以实现无人值守运行。怎么样，是不是很方便？
