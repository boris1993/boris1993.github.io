---
title: "解决命令行中Git显示中文文件名乱码"
source: https://www.boris1993.com/git-commandline-unescape-chinese-characters.html
date: 2020-02-10
updated: 2026-10-10
tags: [Git]
categories: [小技巧]
---

# 解决命令行中Git显示中文文件名乱码

在用Git管理包含中文的文件时，会出现类似这样的“乱码”：

```bash
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
    "\345\271\277\345\221\212\345\220\214\346\255\245\345\271\263\345\217\260\346\216\245\345\217\243\346\226\207\346\241\2432.0.docx.new"

nothing added to commit but untracked files present (use "git add" to track)
```

解决方法是配置Git的全局属性`core.quotepath`为`false`，即执行命令`git config --global core.quotepath false`，然后再执行`git status`，就可以看到中文正常显示了：

```bash
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
    广告同步平台接口文档2.0.docx.new

nothing added to commit but untracked files present (use "git add" to track)
```
