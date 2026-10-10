---
title: "找到并处理Docker容器中的僵尸进程"
source: https://www.boris1993.com/finding-the-docker-container-of-a-zombie-process.html
date: 2024-09-19
updated: 2026-10-10
tags: [Docker, Zombie Process, 僵尸进程]
categories: [小技巧]
---

# 找到并处理Docker容器中的僵尸进程

刚刚登录到我的服务器之后，看到motd提示有一个僵尸进程。本来处理僵尸进程很简单，杀掉它的父进程就行了。但是紧接着我发现这个进程是属于一个Docker容器的，因为我想要更优雅地处理掉它，就顺藤摸瓜找到了对应的容器并将其重启了。这里就记录下我的排查过程以供参考。

因为僵尸进程在`ps`中的状态是`Z`，所以我首先用`ps aux | grep 'Z'`找到这个僵尸进程的PID。此外因为僵尸进程无法被直接杀死，只能杀掉其父进程来将其连根拔起，所以我还需要用`pstree`命令找到它的父进程PID。

```console
$ ps aux | grep 'Z'
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      630024  0.0  0.0      0     0 ?        Z    00:55   0:00 [wget] <defunct>
boris19+ 1180864  0.0  0.0   9696  2332 pts/0    S+   23:52   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox Z
$ pstree -p -s 630024
systemd(1)───containerd-shim(2642178)───node(2642199)───wget(630024)
$ ps aux | grep 2642199
boris19+ 1181119  0.0  0.0   9696  2288 pts/0    S+   23:53   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox 2642199
root     2642199  0.0  0.8 21690116 35956 ?      Ssl  Sep07   3:21 node /app/dist/index.js
```

这里可以看到，僵尸`wget`的父进程是`node`，它的PID是`2642199`，而`node`的父进程是`containerd-shim`，也就是说这是一个Docker容器里的进程。我不确定这时候直接杀掉`node`能不能解决问题。但是老话说，来都来了，那不如继续挖下去。所以我开始尝试去找这个`node`是哪个容器运行的。

在这我稍微走了点弯路。一开始我是想通过搜索`index.js`关键词来找到容器，所以执行了下`docker ps | grep index.js`，但是显然这行不通，不然也不会有这篇文了。在稍微网上冲浪之后，我学到了一个新命令`systemd-cgls`，它可以递归展示出cgroup的内容。因为Docker用到的技术之一就是cgroup，那么想必这就是突破口。在执行了它之后，它打出来了一大片东西，就像这样：

```plaintext
Control group /:
-.slice
├─user.slice 
│ └─user-1000.slice 
│   ├─user@1000.service 
│   │ └─init.scope 
│   │   ├─1180199 /lib/systemd/systemd --user
│   │   └─1180200 (sd-pam)
│   └─session-35890.scope 
│     ├─1180196 sshd: boris1993 [priv]
│     ├─1180374 sshd: boris1993@pts/0
│     ├─1180376 -zsh
│     ├─1182022 systemd-cgls
│     └─1182023 pager
├─init.scope 
│ └─1 /sbin/init
└─system.slice 
  ├─irqbalance.service 
  │ └─941 /usr/sbin/irqbalance --foreground
  ├─docker-27434de50f56b4e096bf5f38bafc76f5c74622c758be1a3f2b9531a3549f4550.scope 
  │ └─3204915 /jellyfin/jellyfin
  ├─docker-adf03dfa49427d2d651cd9101c3033adb00396ed45c390dd6bfda0b9b73eeae3.scope 
  │ └─3775 /watchtower
  ├─open-vm-tools.service 
  │ └─812 /usr/bin/vmtoolsd
  ├─containerd.service 
  │ ├─   1017 /usr/bin/containerd
  │ ├─   2197 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 0e752935d348c3a41d66fe539a06268ee29c6975d1d8f0da41bd0c52b5adf337>
  │ ├─   2479 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 6f5e80ca8977314f3d3a9a47a93509438040a6908bde34cc578cdd3e8263c01a>
  │ ├─   2497 /usr/bin/containerd-shim-runc-v2 -namespace moby -id db3750e57842656c285e9641c4391df926721c94722bfb323789967c3a76d23f>
  │ ├─   2886 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 46520cd2506055c50ef361f50381a65b5077ee7b7d2607e26290d70f8b7292ba>
  │ ├─   3449 /usr/bin/containerd-shim-runc-v2 -namespace moby -id d45b7c4e806f1c0e681f317b548f795ff09eab53bc4d1a6bde05aa1fbb5064ee>
  │ ├─   3451 /usr/bin/containerd-shim-runc-v2 -namespace moby -id adf03dfa49427d2d651cd9101c3033adb00396ed45c390dd6bfda0b9b73eeae3>
  │ ├─   3554 /usr/bin/containerd-shim-runc-v2 -namespace moby -id fd2c2b6dfd77b23c0cdd59d50b33bbf70ab8046bf27ffeb595fbeadadf0a4a99>
  │ ├─   3837 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 6195cb19666ad7c58ce26a115ba4f2ea3b32185acb0c33ff80cbade860693fd0>
  │ ├─   3840 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 559a99dc9580a0ddea8501df71bbce3affb90cf3a91152b8274ae508c20e0f32>
  │ ├─   3901 /usr/bin/containerd-shim-runc-v2 -namespace moby -id b11d850487921e336178dd9267b0f281eea279d0135d1d5b853e63f945246baf>
  │ ├─   3964 /usr/bin/containerd-shim-runc-v2 -namespace moby -id f3f9400aeab6331d55166b1c80cd3e27dc2083664a6126cccf279340ab738364>
  │ ├─   5852 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 722497f0a71df4060151468e590b5a69a49332ba379b17c9504077b18bbd1dfc>
```

好在它的输出就像`less`一样，可以上下卷动，也可以用`ed`命令（就像`vi`一样），那么自然而然，我可以用`/2642199`来找到那一行。

```plaintext
├─docker-ce8f490423d791ed5f6d1aa2b705d797fc3fd2ddc34816d47c228f6fb9a20c63.scope 
│ └─2642199 node /app/dist/index.js
```

好，现在我们知道了容器的ID是`ce8f490423d7`。虽然现在就可以重启它了，但是我的好奇心不允许我这么做，我想知道是哪个容器。所以

```plaintext
$ docker ps | grep ce8f490423d
ce8f490423d7   boris1993/qinglong-bot:latest                                      "docker-entrypoint.s…"   12 days ago     Up 12 days (healthy)   0.0.0.0:3001->3000/tcp, :::3001->3000/tcp                                                                                     qinglong-bot
```

啊好吧，竟然是我那个青龙的机器人。这里面我只用了`wget`来做容器的liveness check，我不理解这么简单的操作怎么就能整出一个僵尸进程来。算了，以后有空再分析吧。重启容器，杀掉僵尸，睡觉。

```plaintext
$ docker restart ce8f490423d
ce8f490423d
```

这时候再用`ps aux | grep 'Z'`就看不到有僵尸进程，也就说明处置成功了。
