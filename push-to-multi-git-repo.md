---
title: "Git连接多个远程仓库"
source: https://www.boris1993.com/push-to-multi-git-repo.html
date: 2017-12-12
updated: 2026-10-10
tags: [Git]
categories: [小技巧]
---

# Git连接多个远程仓库

有时候我们可能会需要push到多个远程仓库，比如同时链接多个代码托管平台的账号，那么可以参考本文所述的方法配置。

保险起见在操作之前请先做好备份工作，毕竟数据无价。

# 方法1 - 添加多个远程仓库

比如要链接两个 Github 仓库，分别是 github1 和 github2，那么：

```bash
# 添加 github1
git remote add github1 https://github.com/username/github1.git

# 添加 github2
git remote add github2 https://github.com/username/github2.git

# 提交到 github1
git push github1 master

# 提交到 github2
git push github2 master

# 从 github1 更新
git pull github1 master

# 从 github2 更新
git pull github2 master
```

# 方法2 - 添加同名多个远程仓库

```bash
# 添加一个远程仓库
git remote add origin https://github.com/username/github1.git

# 然后分别设定push URL
git remote set-url --add --push origin https://github.com/username/github1.git
git remote set-url --add --push origin https://github.com/username/github2.git

# 检查远程仓库配置
git remote -v

# 若配置正确，则结果应当包含一个fetch路径和两个push路径

# 向所有远程仓库推送
git push origin master
```

# 方法3 - 直接修改.git/config文件

用文本编辑器打开本地仓库的 .git/config 文件，然后修改其中的远程仓库配置

```ini
# 假设当前的远程仓库名为 origin
[remote "origin"]
    url = https://github.com/username/github1.git
    fetch = +refs/heads/*:refs/remotes/github/*
    pushurl = https://github.com/username/github1.git
    pushurl = https://github.com/username/github2.git
```

然后直接使用

```bash
git push origin master
```

即可提交至所有版本库
