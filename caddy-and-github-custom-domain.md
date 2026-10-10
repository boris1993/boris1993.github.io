---
title: "记一个Caddy和GitHub自定义域名的坑"
source: https://www.boris1993.com/caddy-and-github-custom-domain.html
date: 2018-08-10
updated: 2026-10-10
tags: [Caddy, GitHub, Custom domain]
categories: [瞎折腾]
---

# 记一个Caddy和GitHub自定义域名的坑

最近心血来潮，想给这个博客在GitHub上面的页面绑个自定义域名，结果无意中发现了一个坑。

# 前情提要

如关于页面所见，这个博客是同时放在GitHub Pages和我的服务器上面的。我的服务器上面呢，是用Caddy的`git`插件监听了一个`WebHook`来实现同步更新的。

在我绑定Custom domain之前，`Caddy`的自动更新一直在默默正常工作着。但就在我绑了Custom domain之后，我发现，Caddy没能成功拉取最新版本的仓库。

# 追踪线索

首先使用排除法，肯定不是GitHub的问题。那就看一下Caddy的日志里面有没有什么线索吧。

```text
Aug 09 14:44:42 vps caddy[4516]: 2018/08/09 14:44:42 Received pull notification for the tracking branch, updating...
Aug 09 14:44:43 vps caddy[4516]: From https://github.com/boris1993/boris1993.github.io
Aug 09 14:44:43 vps caddy[4516]:  * branch            master     -> FETCH_HEAD
Aug 09 14:44:43 vps caddy[4516]:  + 3d5ecea...204143b master     -> origin/master  (forced update)
Aug 09 14:44:43 vps caddy[4516]: *** Please tell me who you are.
Aug 09 14:44:43 vps caddy[4516]: Run
Aug 09 14:44:43 vps caddy[4516]:   git config --global user.email "you@example.com"
Aug 09 14:44:43 vps caddy[4516]:   git config --global user.name "Your Name"
Aug 09 14:44:43 vps caddy[4516]: to set your account's default identity.
Aug 09 14:44:43 vps caddy[4516]: Omit --global to set the identity only in this repository.
Aug 09 14:44:43 vps caddy[4516]: fatal: unable to auto-detect email address (got 'www-data@vps.(none)')
Aug 09 14:44:43 vps caddy[4516]: 2018/08/09 14:44:43 exit status 128
```

鞥？啥时候`git pull`也要提供用户名和邮箱了？

随手往上面翻了翻，看见了点更有意思的东西：

```text
Aug 10 16:45:46 vps caddy[11022]: 2018/08/10 16:45:46 Received pull notification for the tracking branch, updating...
Aug 10 16:45:47 vps caddy[11022]: From https://github.com/boris1993/boris1993.github.io
Aug 10 16:45:47 vps caddy[11022]:  * branch            master     -> FETCH_HEAD
Aug 10 16:45:47 vps caddy[11022]:    3a305c6..b57b257  master     -> origin/master
Aug 10 16:45:47 vps caddy[11022]: Updating 3a305c6..b57b257
Aug 10 16:45:47 vps caddy[11022]: Fast-forward
Aug 10 16:45:47 vps caddy[11022]:  CNAME | 1 +
Aug 10 16:45:47 vps caddy[11022]:  1 file changed, 1 insertion(+)
Aug 10 16:45:47 vps caddy[11022]:  create mode 100644 CNAME
Aug 10 16:45:47 vps caddy[11022]: 2018/08/10 16:45:47 https://github.com/boris1993/boris1993.github.io.git pulled.
```

新增了个叫`CNAME`的文件？这是啥玩意？得，看看里面写了啥。

```bash
$ cat CNAME
blog2.boris1993.tk
$
```

这……不是我刚绑的那个自定义域名么……原来是这么实现的……

好吧，这样一来，问题就清楚了。

# 结案

其实这个问题，是这么回事：

在配置了自定义域名之后，GitHub会往仓库里放一个名为`CNAME`的文件，而我在用hexo提交的时候，我本地完全没有关于这个文件的任何记录，导致远端仓库的CNAME文件又丢了，而在Caddy更新的时候，怀疑Caddy在进行merge操作，merge操作需要用户提供用户名和邮箱，但是运行Caddy的`www-data`用户下没有这两个配置，于是就导致了上面的错误。

至于解决方案嘛，要么就往博客的源码里面放一个名为`CNAME`文件并且保证内容正确，要么就干脆不配置自定义域名了。
