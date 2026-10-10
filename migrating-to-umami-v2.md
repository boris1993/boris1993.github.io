---
title: "从umami v1迁移到v2"
source: https://www.boris1993.com/migrating-to-umami-v2.html
date: 2023-04-18
updated: 2026-10-10
tags: [umami, analytics]
categories: [瞎折腾]
---

# 从umami v1迁移到v2

昨天收到Vercel发的报警邮件，说我自己部署的分析平台`umami`部署失败。检查之后发现是`umami`有个大版本升级，而升级脚本失败导致新版本部署失败了。那既然报错了，就修呗。

虽然不是什么复杂的问题，但还是记录一下好了，也当是水一篇文。

## 问题的根源

一开始发现自动迁移不成功，那么按照官方的迁移文档，我手动运行了迁移脚本，但是得到了这样的错误信息：`db error: ERROR: must be owner of table _prisma_migrations`。看起来是表的权限问题，`_prisma_migrations`这个表的`owner`必须是我用来执行脚本的用户。

搜了一下，`PostgreSQL`里面每个表都有一个所有者，而我一开始是用`postgres`这个用户初始化的数据库，所以这些表的所有者都是`postgres`，之前没有问题，只是因为我给umami的用户读写这些表的授权了。

## 修复数据库

首先执行了下`select schemaname, tablename, tableowner from pg_tables where schemaname = 'umami'`，果不其然这些表的owner都是`postgres`。

于是尝试执行了下`alter table umami_analytics._prisma_migrations owner to umami_analytics`，但是又报错`ERROR: must be member of role umami_user`。好么，我一直以为`postgres`用户跟MySQL的`root`一样是超管，结果PostgreSQL世界里面人人生而平等？好吧，你要权限那我就给你授权，`grant postgres to umami_user`。

授权之后，重新执行`alter table umami_analytics._prisma_migrations owner to umami_analytics`，发现成功了，再执行umami的迁移脚本后，发现错误信息变成了`db error: ERROR: must be owner of table account`。看来，上面的解决方案奏效了，接下来就是把剩下的表的所有者全都改过来。

```sql
alter table umami_analytics._event_old owner to umami_analytics;
alter table umami_analytics._prisma_migrations owner to umami_analytics;
alter table umami_analytics.account owner to umami_analytics;
alter table umami_analytics."event" owner to umami_analytics;
alter table umami_analytics.event_data owner to umami_analytics;
alter table umami_analytics.pageview owner to umami_analytics;
alter table umami_analytics."session" owner to umami_analytics;
alter table umami_analytics.website owner to umami_analytics;

select schemaname, tablename, tableowner from pg_tables where schemaname = 'umami_analytics';
```

可以看到，umami相关的表的所有者都正确了，回到Vercel，重新运行失败的部署，发现还是报错。无奈，又试了试手动迁移脚本，竟然成功了，这时候数据库肯定是v2的了，再到Vercel重新部署，这次就成功了。

注意脚本执行到最后会问要不要删掉v1的表，记得不要删。我发现在删掉v1的旧表之后，Vercel的部署又会出新的问题。（我为了验证到底是我的问题还是脚本的问题，回滚了两三次生产数据库。也就是这个数据不重要我才敢这么折腾，好孩子不要学我乱搞生产数据库哦～）

## 更新博客的配置

NexT主题是内建了对umami的支持的，但是需要手动指定脚本的位置。根据迁移文档的提示，修正`_config.next.yml`中`umami.script_url`，重新部署博客即可。
