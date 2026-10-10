---
title: "分环境为Git指定单独的配置"
source: https://www.boris1993.com/specify-different-git-config-for-different-environments.html
date: 2019-09-18
updated: 2026-10-10
tags: [Git]
categories: [小技巧]
---

# 分环境为Git指定单独的配置

有的小伙伴，在公司需要使用公司的Git来干活，同时自己也有些托管在GitHub等其他仓库的代码，在这种情况下会有一个烦恼，就是怎么样在多个仓库之间使用不同的配置。比如在公司仓库和个人仓库之间使用不同的身份信息，或者在公司仓库和个人仓库之间，使用不同的GPG签名。

在`2.13`版本之前，大概你就只能单独为每一个仓库单独设定这些信息了，但是，在`2.13`这个版本中，Git引入了一个名为“按条件引入”(Conditional includes)的功能。这个功能允许用户通过指定一定的条件，来使Git从不同的配置文件中取得配置项。

## 准备工作

“按条件引入”功能可以根据**目录**或者**分支**来决定使用哪个配置，显然根据分支无法区分工作环境和私人环境，所以首先需要分别为工作环境和私人环境创建各自的目录。比如将目录划分为这样：

```text
projects
├── work     <== 工作相关
└── personal <== 私人项目
```

## 拆分配置

接下来，需要为全局、工作、私人分别编写配置文件。本文中，全局配置文件`.gitconfig`位于`$HOME`下，各环境的配置文件位于`~/.config/git`下。

### 全局配置文件

```ini
[core]
  excludesfile = ~/.gitignore_global
  editor = vim
[includeIf "gitdir:~/projects/work/"]
  path = ~/.config/git/gitconfig-work
[includeIf "gitdir:~/projects/personal/"]
  path = ~/.config/git/gitconfig-personal
```

第4行和第6行的`includeIf`段，就是“按条件引入”的配置。在这里需要注意这几点：

-   `gitdir:`参数用于匹配本地Git仓库的路径。
    -   如果文件系统是大小写不敏感的，并且路径同时包含大写和小写字母，那么需要使用`gitdir/i:`来匹配。
    -   如果要匹配某个目录下面的所有子目录，那么在路径最后需要加上`/`或者`/**`。（实际上，Git会自动在末尾的`/`后面附加`**`）
    -   反之，如果只要匹配到某一个目录，而不递归包含其下面的所有子目录，那么路径末尾就不要有`/`。
-   `path`指定了在满足条件时要使用的配置文件

综上所述，位于`~/projects/work/`下的所有Git仓库，都套用`~/.config/git/gitconfig-work`中的配置；位于`~/projects/personal/`下的所有Git仓库，都套用`~/.config/git/gitconfig-personal`中的配置。

### 私人环境和工作环境的配置文件

这两个环境各自的配置文件就没有什么新鲜的东西了，就只有面向各个环境的身份信息、GPG签名信息等。

```ini
[user]
  signingkey = 1122334455667788
  name = Your Name
  email = test@example.com
[commit]
  gpgsign = true
  template = ~/.stCommitMsg
[gpg]
  program = /usr/local/bin/gpg
```

## 检查配置

写完了配置文件，还是要检查一下是不是生效了的。首先，在一个不包含任何Git仓库的位置执行`git config -l`，返回的信息中就只有全局配置中的值。

```text
core.excludesfile=~/.gitignore_global
core.editor=vim
includeif.gitdir:~/projects/mininglamp/.path=~/.config/git/gitconfig-mininglamp
includeif.gitdir:~/projects/personal/.path=~/.config/git/gitconfig-default
```

然后，进入私人环境的某个Git仓库，再执行`git config -l`，就可以看到私人环境相关的配置文件被引入了。（仓库自己的配置其实也会被显示出来，这里为了减小篇幅将其略掉了）

```text
core.excludesfile=~/.gitignore_global
core.editor=vim
includeif.gitdir:~/projects/mininglamp/.path=~/.config/git/gitconfig-mininglamp
includeif.gitdir:~/projects/personal/.path=~/.config/git/gitconfig-default
user.signingkey=1122334455667788
user.name=Your Name
user.email=test@example.com
commit.gpgsign=true
commit.template=~/.stCommitMsg
gpg.program=/usr/local/bin/gpg
```

## 参考资料

-   [Git官方文档的`Includes`部分](https://git-scm.com/docs/git-config#_includes)
-   Git 2.13 有趣的新功能介绍 - GitHub
-   Conditional Includes For Git Config
