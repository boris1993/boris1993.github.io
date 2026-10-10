---
title: "记一次抢救被force push的GitHub仓库"
source: https://www.boris1993.com/rescuing-my-github-repo-from-force-push.html
date: 2022-10-17
updated: 2026-10-10
tags: [GitHub, git, force push, 恢复GitHub仓库]
categories: [瞎折腾]
---

# 记一次抢救被force push的GitHub仓库

就在刚刚，我一个误操作，在没有本地备份的前提下，force push了一个GitHub上的仓库。万幸最后恢复成功，数据拿回来了。惊魂未定之余，在此记录我的抢救过程以供参考。

## 前景提要

在闲逛GitHub的时候，发现了一个叫snk的项目，可以在我的profile readme里面放个贪吃蛇，遂照着它的example with cron job抄了一个workflow过来。但是这里我自作聪明地想着把东西全放在`master`上，就把38行的`target_branch`改成了`master`。结果一运行吓一跳，我的`README.md`没了，仓库只剩下`snk`生成的svg文件。这不行啊，我花了好久的时间才整出来的东西，不能说没就没啊！于是赶紧开始网上冲浪，看怎么抢救被force push的repo。

## 恢复过程

首先我要大力感谢这个Gist，我就是参考这里的做法成功恢复了这个repo的。

### 找到上一次commit的记录

首先，要通过`https://api.github.com/repos/:owner/:repo/events`这个API找到上次提交的sha。

```bash
-> % curl -u boris1993 https://api.github.com/repos/boris1993/boris1993/events
Enter host password for user 'boris1993':
[
  {
    "id": "24633558565",
    "type": "PushEvent",
    "actor": {
      "id": 4367313,
      "login": "boris1993",
      "display_login": "boris1993",
      "gravatar_id": "",
      "url": "https://api.github.com/users/boris1993",
      "avatar_url": "https://avatars.githubusercontent.com/u/4367313?"
    },
    "repo": {
      "id": 297097347,
      "name": "boris1993/boris1993",
      "url": "https://api.github.com/repos/boris1993/boris1993"
    },
    "payload": {
      "push_id": 11349267173,
      "size": 1,
      "distinct_size": 1,
      "ref": "refs/heads/master",
      "head": "98364ce80ec5bbcdb6dc6f8d2239de2256ede487",
      "before": "32276fc643c6b34fee48f46363cfb6a44327cbe4",
      "commits": [
        {
          "sha": "98364ce80ec5bbcdb6dc6f8d2239de2256ede487",
          "author": {
            "email": "41898282+github-actions[bot]@users.noreply.github.com",
            "name": "github-actions[bot]"
          },
          "message": "Deploy to GitHub pages",
          "distinct": true,
          "url": "https://api.github.com/repos/boris1993/boris1993/commits/98364ce80ec5bbcdb6dc6f8d2239de2256ede487"
        }
      ]
    },
    "public": true,
    "created_at": "2022-10-17T05:44:11Z"
  },
  {
    "id": "24633554175",
    "type": "PushEvent",
    "actor": {
      "id": 4367313,
      "login": "boris1993",
      "display_login": "boris1993",
      "gravatar_id": "",
      "url": "https://api.github.com/users/boris1993",
      "avatar_url": "https://avatars.githubusercontent.com/u/4367313?"
    },
    "repo": {
      "id": 297097347,
      "name": "boris1993/boris1993",
      "url": "https://api.github.com/repos/boris1993/boris1993"
    },
    "payload": {
      "push_id": 11349264873,
      "size": 1,
      "distinct_size": 1,
      "ref": "refs/heads/master",
      "head": "32276fc643c6b34fee48f46363cfb6a44327cbe4",
      "before": "b0ab0263c0693122ae8069c95526e13b7336483f",
      "commits": [
        {
          "sha": "32276fc643c6b34fee48f46363cfb6a44327cbe4",
          "author": {
            "email": "boris1993@live.cn",
            "name": "Boris Zhao"
          },
          "message": "Create generate_snake_animation.yml",
          "distinct": true,
          "url": "https://api.github.com/repos/boris1993/boris1993/commits/32276fc643c6b34fee48f46363cfb6a44327cbe4"
        }
      ]
    },
    "public": true,
    "created_at": "2022-10-17T05:43:50Z"
  },
  {
    "id": "23024411036",
    "type": "PushEvent",
    "actor": {
      "id": 4367313,
      "login": "boris1993",
      "display_login": "boris1993",
      "gravatar_id": "",
      "url": "https://api.github.com/users/boris1993",
      "avatar_url": "https://avatars.githubusercontent.com/u/4367313?"
    },
    "repo": {
      "id": 297097347,
      "name": "boris1993/boris1993",
      "url": "https://api.github.com/repos/boris1993/boris1993"
    },
    "payload": {
      "push_id": 10514884570,
      "size": 1,
      "distinct_size": 1,
      "ref": "refs/heads/master",
      "head": "b0ab0263c0693122ae8069c95526e13b7336483f",
      "before": "660e4d3896eb523d703464f89112a2eea07ee309",
      "commits": [
        {
          "sha": "b0ab0263c0693122ae8069c95526e13b7336483f",
          "author": {
            "email": "boris1993@live.cn",
            "name": "Boris Zhao"
          },
          "message": "Update README.md",
          "distinct": true,
          "url": "https://api.github.com/repos/boris1993/boris1993/commits/b0ab0263c0693122ae8069c95526e13b7336483f"
        }
      ]
    },
    "public": true,
    "created_at": "2022-07-22T08:05:06Z"
  }
]
```

从接口的返回可以看到，截至现在一共有三次push，仓库被覆盖就是发生在最上面的一次push中，而接下来的一个就是我添加workflow的那一次push。

### 从上一次的push记录中找回数据

找到push记录，那么就好办了。点击添加workflow那次push中的`url`，在浏览器中会打开一个新页面，返回的JSON中是这次push的详细信息。这里我们要找的是`html_url`这个字段。点开这个字段里面的链接，会进入GitHub里面，看到这次commit的diff。然后点击`Browse files`按钮，就能看到当时的文件了。这时候还等什么？赶紧下载啊！文件少的话复制内容回来就行，文件多的话，点开`Code`按钮，`Download ZIP`就好啦。

![Files at that commit](https://blog-static.boris1993.com/rescuing-my-github-repo-from-force-push/files_at_that_commit.png)

## 后记

一定要做备份啊！
