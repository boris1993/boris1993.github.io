---
title: "记一次修复生产环境中nginx出现的SNI相关的问题"
source: https://www.boris1993.com/resolving-nginx-sni-issue.html
date: 2021-11-21
updated: 2026-10-10
tags: [SNI, nginx, OpenResty]
categories: [学知识]
---

# 记一次修复生产环境中nginx出现的SNI相关的问题

前段时间，我司出现了一次生产事故，调查后发现是当时的OpenResty配置不兼容SNI导致的。在这里我也记录一下整件事的排查过程，以及解决方法，供遇到类似问题的同志们参考。

# 事故症状

某天开始，我司的OpenResty日志中大量出现SSL握手失败的错误，并影响了正常的业务。查看OpenResty日志，看到有大量这样子的报错：

```plaintext
2021/10/19 20:51:30 [warn] 16776#16776: *1110324 upstream server temporarily disabled while SSL handshaking to upstream, client: [MASKED], server: localhost, request: "GET /endpoint/to/the/api?query=param HTTP/1.1", upstream: "https://MASKED:443/endpoint/to/the/api?query=param", host: "MASKED"

2021/10/19 20:51:30 [error] 16776#16776: *1110324 SSL_do_handshake() failed (SSL: error:14094438:SSL routines:ssl3_read_bytes:tlsv1 alert internal error:SSL alert number 80) while SSL handshaking to upstream, client: MASKED, server: localhost, request: "GET /endpoint/to/the/api?query=param HTTP/1.1", upstream: "https://MASKED:443/endpoint/to/the/api?query=param", host: "MASKED"
```

# 调查中的弯路

知道了是SSL握手失败导致的问题，那么当然接下来就开始调查为什么会握手失败。是解析配置出错？还是证书出现问题？

按照一直的经验，我决定先用`nslookup`检查一下DNS解析。因为保密和时间问题，我就不把`nslookup`的输出放在这里了。简而言之，再出现问题之前，我司的域名都是通过`CNAME`记录解析到Akamai的Edge节点上的，但现在，却直接用`A`记录解析到了一个IP上，这让我感觉很奇怪。同时，我为了确认，又用`openssl`命令连接了一下解析出来的IP，看它会返回什么证书信息，可出乎意料，啥也没有。

我感觉不对劲，于是联系了NetOps组。这时候，NetOps点出了这篇博文的主题，SNI。

他说，给`openssl`命令加一个`-servername`参数，把目标服务器的域名放上去。我一试，好使了，Akamai返回了正确的证书信息。

那么问题来了，解析没问题，那就是我有问题了。但问题出在哪呢？

# SNI是什么

在继续之前，我想先简单介绍一下，什么是SNI。

根据维基百科词条服务器名称指示中的说法：

> 基于名称的虚拟主机允许多个DNS主机名由同一IP地址上的单个服务器（通常为Web服务器）托管。为了实现这一点，服务器使用客户端提供的主机名作为协议的一部分（对于HTTP，名称显示在主机头中）。但是，当使用HTTPS时，TLS握手发生在服务器看到任何HTTP头之前。因此，服务器不可能使用HTTP主机头中的信息来决定呈现哪个证书，并且因此只有由同一证书覆盖的名称才能由同一IP地址提供。  
> ……  
> 实际上，这意味着对于安全浏览来说，HTTPS服务器只能是每个IP地址服务一个域名（或一组域名）。为每个站点分配单独的IP地址会增加托管成本，因为对IP地址的请求必须为区域互联网注册机构提供证据而且现在IPv4地址已用尽。  
> ……  
> 客户端在SNI扩展中发送要连接的主机名称，作为TLS协商的一部分。这使服务器能够提前选择正确的主机名称，并向浏览器提供相应TLS证书。从而，具有单个IP地址的服务器可以在获取公共证书不现实的情况下提供一组域名的TLS连接。

也就是说，在握手的时候，我需要预先提供我要访问的网站的域名，这样服务器才会把正确的证书返回给我。而上面说的`openssl`命令中的`-servername`参数就是做了这件事。

# 无心插柳，柳暗花明

就在我拿着各种关键词Google的时候，一篇文章引起了我的注意。

文章里描述的问题也是在OpenResty中出现了SSL握手失败，同样作者也在`proxy_pass`中用了`upstream`。作者做了一个测试，如果在`proxy_pass`中直接写上游的域名，就没有问题，但是一旦用`upstream`，就会出现握手失败。那么问题一定出现在`upstream`导致的某种行为变化。

然后作者发现，在用域名的时候，OpenResty的变量`$proxy_host`存放的就是域名，可在用`upstream`的时候，这里面就变成了那个`upstream`的名字。

看到这，我知道了，这应该就是我这个问题的解决方案。

# 动手解决问题

首先我先展示一下修复前的OpenResty的一部分配置：

```nginx
location /path/to/endpoint {
    include /etc/nginx/conf.d/proxy.common;
    proxy_set_header Host api.$DOMAIN_FOR_GCP.com;
    proxy_pass https://gcp-https/path/to/the/endpoint/on/server;
}
```

可见，如果按照这个配置，那么我发给Akamai的域名就是`gcp-https`，而不是正确的`api.mycompany.com`。

所以，根据那篇文章，以及参照OpenResty的手册，我在配置中增加了一条`proxy_ssl_name`指令，并将其配置为实际的后端服务的域名。

```nginx
location /path/to/endpoint {
    include /etc/nginx/conf.d/proxy.common;
    proxy_set_header Host api.$DOMAIN_FOR_GCP.com;
    # THIS ONE
    proxy_ssl_name api.$DOMAIN_FOR_GCP.com;
    proxy_pass https://gcp-https/path/to/the/endpoint/on/server;
}
```

可部署新配置之后，问题并未解决，SSL握手失败的问题依旧存在。

然后我注意到，那篇文章中还出现了一个指令`proxy_ssl_server_name on;`。莫非，我们的OpenResty里面干脆没启用SNI支持？

在终端里dump了一下当前的配置，果然没有显式指定`proxy_ssl_server_name`的值，而默认情况下这个是被关闭的。那好办，我在OpenResty的全局配置中把它打开就好了。

然后再次部署测试，发现再没有SSL握手失败的问题，测试环境中业务也恢复了正常。火速打包上线生产环境，事故解决。
