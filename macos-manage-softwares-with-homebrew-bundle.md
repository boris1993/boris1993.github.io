---
title: "使用Homebrew Bundle管理电脑上安装的软件"
source: https://www.boris1993.com/macos-manage-softwares-with-homebrew-bundle.html
date: 2024-03-18
updated: 2026-10-10
tags: [macOS, Homebrew]
categories: [瞎折腾]
---

# 使用Homebrew Bundle管理电脑上安装的软件

今天看到一篇推文说Homebrew也支持类似`package.json`的方式来管理电脑上安装的应用，感觉这玩意挺实用的，像我上回换电脑还是照着`brew list`的输出再一个个装，有了它的话就可以一键安装了。而且除了Homebrew之外，它还可以导出通过App Store安装的软件和VS Code插件。那么，就小小折腾一下。

首次运行`brew bundle`的时候它就会自动安装了。如果要导出App Store安装的程序的话，那么还需要安装`mas`这个Formula。

然后就可以执行`brew bundle dump`，它会在当前目录下生成一个`Brewfile`文件，里面就是你电脑上目前装的软件，内容就像这样：

```ruby
# 各个Homebrew仓库
tap "beeftornado/rmtree"
tap "bell-sw/liberica"
tap "homebrew/autoupdate"
tap "homebrew/bundle"
tap "homebrew/cask-fonts"
tap "homebrew/cask-versions"

# 通过Homebrew安装的Formula
brew "cloudflared"
brew "curl"
brew "git"

# 通过Homebrew安装的Cask
cask "1password"
cask "alfred"
cask "altserver"
cask "arc"
cask "bartender"

# 通过App Store安装的软件
mas "DaVinci Resolve", id: 571213070
mas "Eul", id: 1541991958
mas "Termius", id: 1176074088
mas "WireGuard", id: 1451685025
mas "Xnip", id: 1221250572

# VS Code插件
vscode "ms-python.debugpy"
vscode "ms-python.python"
vscode "redhat.vscode-yaml"
```

我的这个Brewfile算是简单的，它还支持配置安装参数等，具体可以参考Homebrew Bundle的实例。

如果在导出的时候在当前目录已经存在一个`Brewfile`，那么它会报错说文件已存在。这时候可以加`--force`参数强制导出，要在cron里定时导出的话这个参数会很有用。此外如果你想要每个软件的简介，那么还可以加上`--describe`参数，出来的`Brewfile`就会是这样的：

```ruby
# Mac App Store command-line interface
brew "mas"
# Java-based project management
brew "maven"
# Menu bar icon organiser
cask "bartender"
```

不过`--describe`参数只对`brew`和`cask`条目生效，`tap`、`mas`和`vscode`条目都是没有描述的。

我试用后发现，似乎`mas`这部分会把你已经卸载的软件也列出来，而第二次再dump的时候又另外发现了一个之前没列出来而已经通过App Store安装的程序，所以这部分可能需要在导出之后自己再人工校验一下。

要按照`Brewfile`安装软件的话，只需要cd到`Brewfile`所在位置然后执行`brew bundle`就会自动开始安装。

如果要把不在`Brewfile`中的软件删除，那么可以运行`brew bundle cleanup`命令。

此外，运行`brew bundle check`可以检查哪些软件在`Brewfile`里列出来了，但是还没安装。但是我用刚刚导出的`Brewfile`检查却报了`brew bundle can't satisfy your Brewfile's dependencies.Satisfy missing dependencies with ``brew bundle install``.`这样的消息，我就有点怀疑这个检查到底准不准……
