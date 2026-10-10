---
title: "用裸git仓库管理点文件的最佳方式"
source: https://www.boris1993.com/translation-dotfiles-best-way-to-store-in-a-bare-git-repository.html
date: 2024-05-09
updated: 2026-10-10
tags: [git, dotfiles, 点文件]
categories: [翻译]
---

# 用裸git仓库管理点文件的最佳方式

前些天在Atlassian的文档中看到一篇关于管理点文件（即`.bashrc`之类以点开头的文件）的文章《Dotfiles: Best way to store in a bare git repository》，感觉很有参考价值，遂决定翻译出来，希望能帮到更多的人。

以下内容除特别注明外，皆翻译自原文。我亦不对内容做任何的担保，并不对任何可能产生的后果（包括但不限于文件丢失）负责。

* * *

免责声明：标题有些夸大其词了，而且针对这个问题也有其他好用的解决方案。但我确实觉得这也是一个优雅的技巧。

最近我在Hacker News中看到一篇帖子，是讨论人们怎么管理他们的点文件的。在这篇帖子中，用户`StreakyCobra`分享了一个他的很优雅的配置，而且我觉得非常的靠谱。正巧我也在开始用同样的技巧来管理我的系统。这个技巧也只有一个前置条件：你安装了Git。

按他的说法，这个技巧：

> 不需要别的工具，不需要创建符号链接，文件都被版本控制系统跟踪，你可以用不同的分支来管理不同的系统，在全新的系统中你也可以轻松复用你的配置。

这个技巧包括两个部分：用一个单独的文件夹（如`$HOME/.cfg`或`$HOME/.myconfig`）作为裸Git仓库；和一个专门用来操作这个仓库的命令别名（译者注：`alias`）。

## 从零开始

如果你还没有用一个Git仓库来跟踪你的配置文件，那么你可以从下面这几个简单的命令开始，一步步实现这个技巧。

```bash
git init --bare $HOME/.cfg
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
config config --local status.showUntrackedFiles no
echo "alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'" >> $HOME/.bashrc
```

-   第一条命令会创建一个用来跟踪文件的裸Git仓库`~/.cfg`（配置仓库）
-   接下来创建一个命令别名`config`，在我们要操作配置仓库时就会用这个命令，而不是`git`命令
-   设定一个仅对配置仓库生效的设定 —— 不显示未追踪的文件，这样在我们执行`config status`等命令的时候，我们不想跟踪的文件就不会以`untracked`状态被显示出来
-   此外我们可以把这个别名的声明添加到`.bashrc`里面，方便以后使用

我把上面这些命令集合成了一个代码片段放到了Bitbucket上，并给它做了一个短链接，所以你可以用下面的命令一键执行：

```bash
curl -Lks http://bit.do/cfg-init | /bin/bash
```

（译者注：在执行来自网络的代码之前，一定记得先看看内容。闭著眼执行远程命令是一件很危险的事，因为你不知道这个链接的内容是否还正确，比如曾经课本上本来指向《历朝历代咏武侯诗词大全》的网址，在域名过期之后被黄网搞去了……）

在配置完成后，`$HOME`目录下的文件就可以用刚刚创建的`config`别名来进行版本控制了，比如：

```bash
config status
config add .vimrc
config commit -m "Add .vimrc"
config add .bashrc
config commit -m "Add .bashrc"
config push
```

## 把这些点文件安装到新系统（或迁移到本系统）

如果你已经用Git仓库管理你的配置或者点文件，那你可以跟着下面的步骤，来把这个配置迁移到新的系统：

-   首先确认你已经配置好了命令别名

```bash
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
```

-   在克隆你的配置仓库前，记得让git忽略它，以避免各种奇怪的问题

```bash
echo ".cfg" >> .gitignore
```

-   然后你就可以把你的点文件克隆到一个存在于“点目录”的裸仓库中

```bash
git clone --bare <git-repo-url> $HOME/.cfg
```

-   在当前shell会话中定义好`config`这个别名

```bash
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
```

-   从裸仓库中把配置checkout到`$HOME`

```bash
config checkout
```

这一步可能会报这样的错误

```text
error: The following untracked working tree files would be overwritten by checkout:
    .bashrc
    .gitignore
Please move or remove them before you can switch branches.
Aborting
```

这是因为你的`$HOME`目录可能已经有这些文件，而checkout操作可能会让它们被覆盖。解决方法也很简单：这些文件要是有用，那就备份出来，没用那就删了。你可以用我这个比较粗糙的命令，来一键把有冲突的文件移动到备份目录：

```bash
mkdir -p .config-backup && \
config checkout 2>&1 | egrep "\s+\." | awk {'print $1'} | \
xargs -I{} mv {} .config-backup/{}
```

然后重新执行`config checkout`

-   为配置仓库设定不显示未跟踪的文件

```bash
config config --local status.showUntrackedFiles no
```

-   到此就完成了，接下来你就可以和之前一样用`config`命令来管理你的点文件了。

同样，我也提供了一个一键脚本，把它作为代码片段放在了Bitbucket上。你可以用这个命令来执行：

```bash
curl -Lks http://bit.do/cfg-install | /bin/bash
```

为了文章的完整起见，这是我最终得到的脚本（已经在很多个全新启动的Alpine Linux容器中做过测试）

```bash
git clone --bare https://bitbucket.org/durdn/cfg.git $HOME/.cfg
function config {
   /usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME $@
}
mkdir -p .config-backup
config checkout
if [ $? = 0 ]; then
  echo "Checked out config.";
else
  echo "Backing up pre-existing dot files.";
  config checkout 2>&1 | egrep "\s+\." | awk {'print $1'} | xargs -I{} mv {} .config-backup/{}
fi;
config checkout
config config status.showUntrackedFiles no
```

（译者注：原文的`if-else`部分似乎缩进有问题，我调整了下格式，未对脚本内容做改动）

## 结语

我希望你们能觉得这个管理点文件的小技巧能产生帮助。如果你有兴趣的话，可以到这里看我的点文件。另外别忘了关注@durdn或我那炫酷的小组@atlassiandev。
