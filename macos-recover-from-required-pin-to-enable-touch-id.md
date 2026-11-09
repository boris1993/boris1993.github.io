---
title: "记一次在MacBook上瞎折腾YubiKey把自己锁在外面的救援"
source: https://www.boris1993.com/macos-recover-from-required-pin-to-enable-touch-id.html
date: 2026-10-22
updated: 2026-11-09
tags: [macOS, MacBook, YubiKey]
categories: [瞎折腾]
---

# 记一次在MacBook上瞎折腾YubiKey把自己锁在外面的救援

前两天拿着YubiKey瞎折腾玩，给我的MacBook配上了用它登录，配完了又担心将来忘了这件事给自己找麻烦，于是又解绑了。尽管当时用`sc_auth list`确认了账户已经没有绑定的智能卡，但刚刚打开电脑的时候，却发现macOS要我输入PIN来启用Touch ID。在反复试了好几个我确定正确的PIN都不对之后，只好重启进恢复模式手动改配置绕过。

> 感谢Claude大师救我于水火，否则我自己查文档去修的话可能得要一下午了🤣

-   重启到恢复模式，进入磁盘工具，把`Macintosh HD - Data`卷挂载上。
-   打开终端，确认你的用户记录文件到位置

```bash
ls "/Volumes/Macintosh HD - Data/private/var/db/dslocal/nodes/Default/users/"
```

-   找到自己的用户配置文件（此处以`myuser`为例），用如下命令修改配置

```bash
# 改东西之前备份是个好习惯
cp myuser.plist myuser.plist.bak

# 同时我也习惯改之前看一眼内容，此时确认了当前状态下没有SmartCardEnforcement这个key
defaults read "/Volumes/Macintosh HD - Data/private/var/db/dslocal/nodes/Default/users/myuser.plist"

# 这里一定要用绝对路径，否则defaults命令会报找不到domain
defaults write "/Volumes/Macintosh HD - Data/private/var/db/dslocal/nodes/Default/users/myuser.plist" SmartCardEnforcement -array-add 2
```

> 稍微解释一下这条命令在干嘛:  
> SmartCardEnforcement配置了用户级的智能卡强制开关，
> 
> > `0 - 遵循系统默认设置，1 - 对该用户强制启用智能卡，2 - 对该用户禁用智能卡强制`
> 
> 在正常运行时可以用命令 `dscl . -append /Users/myuser SmartCardEnforcement 2`来完成同样的事，此时dscl会增加一个名为SmartCardEnforcement的数组并追加一个元素2  
> 上面的 defaults write 就是在手工完成这件事

-   重启电脑，此时在登录界面应该就变回让你用密码登录了
-   登录进去之后，清除配对残留

```bash
# 习惯先看一眼有些啥，我此时查到是空的
sudo sc_auth list -u $(whoami)

# 清除智能卡配对
sudo sc_auth unpair -u $(whoami)
```

-   去掉刚刚手动添加的标记

```bash
sudo dscl . -delete /Users/$(whoami) SmartCardEnforcement
```

-   然后锁屏，用没有登记到Touch ID的指纹连续按5次触发锁定，此时登录界面应该提示需要密码来启动Touch ID，而不是需要PIN
-   至此就救回来了
