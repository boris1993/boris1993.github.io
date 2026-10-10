---
title: "记给Ubuntu的LVM卷扩容"
source: https://www.boris1993.com/expanding-ubuntu-lvm-partition.html
date: 2024-12-09
updated: 2026-10-10
tags: [Ubuntu, LVM]
categories: [小技巧]
---

# 记给Ubuntu的LVM卷扩容

刚刚看到我虚拟机里面的Ubuntu报硬盘空间不足，看了下发现是在安装那会，自动分区就只用了一半的空间，遂着手扩容。在这里记录下操作过程备忘。

看见硬盘空间不足，首先下意识`df -h`了一下，确实满了。

```plaintext
$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  3.5M  389M   1% /run
efivarfs                            64K   41K   19K  70% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv   48G   45G  314M 100% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  266M  1.6G  15% /boot
/dev/sda1                          1.1G  6.2M  1.1G   1% /boot/efi
tmpfs                              392M   12K  392M   1% /run/user/1000
```

但是ESXi里面显示这个虚拟机是有一个100GB的虚拟磁盘的，而`df`只显示出将近50GB，那就是说要么有50GB没分，要么出别的问题了。首先排除最简单的情况，看看是不是有一半的硬盘没分进去。

```plaintext
$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME                      FSTYPE        SIZE MOUNTPOINT                    LABEL
loop0                                  10.1M /snap/canonical-livepatch/282 
loop1                                  10.7M /snap/canonical-livepatch/286 
loop2                                   104M /snap/core/16928              
loop3                                 104.2M /snap/core/17200              
loop4                                  55.7M /snap/core18/2829             
loop5                                  55.4M /snap/core18/2846             
loop6                                    64M /snap/core20/2379             
loop7                                  63.7M /snap/core20/2434             
loop9                                  73.9M /snap/core22/1663             
loop10                                   71M /snap/prometheus/86           
loop11                                 38.8M /snap/snapd/21759             
loop12                                 44.3M /snap/snapd/23258             
loop13                                 73.9M /snap/core22/1722             
sda                                     100G                               
├─sda1                    vfat            1G /boot/efi                     
├─sda2                    ext4            2G /boot                         
└─sda3                    LVM2_member  96.9G                               
  └─ubuntu--vg-ubuntu--lv ext4         48.5G /                             
sr0                                    1024M                               
```

从上面的命令输出可以看到，`sda`这块盘的100GB都认出来了，`sda3`分区也确实分了96.9GB，但是`ubuntu--vg-ubuntu--lv`这个卷只分配了48.5GB，也就是整个LVM的一半。

查阅了教程，了解到可以用`lvdisplay`来检查LVM逻辑卷的空间，并可以用`vgdisplay`检查volume group的空间。遂分别执行。

```plaintext
$ sudo vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <96.95 GiB
  PE Size               4.00 MiB
  Total PE              24818
  Alloc PE / Size       12409 / 48.47 GiB
  Free  PE / Size       12409 / 48.47 GiB
  VG UUID               zaKkwe-YemQ-dRcV-mrhe-v181-0Ixc-0KVaVL

$ sudo pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <96.95 GiB / not usable 3.00 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              24818
  Free PE               12409
  Allocated PE          12409
  PV UUID               3FWWGz-CbIQ-BI9e-vMMd-p0Hu-eWUa-rZSeK8
```

其实看不看都一样，已经知道根源是空间没全部分给这个卷。那么接下来就可以用`lvextend`来扩展指定的卷。我直接把所有的空余空间全都分配进去。

```plaintext
$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv 
  Size of logical volume ubuntu-vg/ubuntu-lv changed from 48.47 GiB (12409 extents) to <96.95 GiB (24818 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

OK，卷扩容成功。但这时候实际的文件系统大小还没变，接下来需要用`resize2fs`扩容文件系统。

```plaintext
$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv 
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 7, new_desc_blocks = 13
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 25413632 (4k) blocks long.
```

完事，这时候再`df -h`就能看到根挂载点有将近100GB的空间了，扩容成功。

```plaintext
$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  3.5M  389M   1% /run
efivarfs                            64K   41K   19K  70% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv   96G   45G   47G  50% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  266M  1.6G  15% /boot
/dev/sda1                          1.1G  6.2M  1.1G   1% /boot/efi
tmpfs                              392M   12K  392M   1% /run/user/1000
```
