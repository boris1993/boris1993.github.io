---
title: "二手MicroServer Gen8折腾记(1/n) - 初始化服务器"
source: https://www.boris1993.com/setting-up-microserver-gen8.html
date: 2022-12-18
updated: 2026-10-10
tags: [二手MicroServer Gen8折腾记, Gen8]
categories: [瞎折腾]
---

# 二手MicroServer Gen8折腾记(1/n) - 初始化服务器

上个月冲动消费，入手了一台HPE ProLiant MicroServer Gen8。其实我早想搞一台正经的服务器来玩了，因为群晖的魔改Linux总是玩的不那么爽。现在心愿实现，开始折腾。

本篇为[二手MicroServer Gen8折腾记](https://www.boris1993.com/tags/%E4%BA%8C%E6%89%8BMicroServer-Gen8%E6%8A%98%E8%85%BE%E8%AE%B0/)的第一篇，记录拿到服务器之后做的一些初始化的工作。

## 重置iLO

到手之后，我发现我把事想简单了，服务器一般只给一个VGA口，而这台也不例外，然而，我这里没有一个能接VGA口的设备。是的，你没看错。不止显示器没有VGA口，连电视也没有。遂果断下单了一个VGA到HDMI转接器。第二天收到快递，兴冲冲的接上，却发现只能显示POST画面，之后无论是正常继续启动，还是进RBSU（ROM based setup utility），电视上都是无信号。事到如此，接显示器这条路算是堵死了。别问我为什么不买个支持VGA的显示器，你自己想想这成本划得来么？

虽然接显示器不行，但是天无绝人之路啊，地球人都知道，正经服务器上都有一个带外管理系统，Gen 8也不例外，带有惠普的iLO（Integrated Lights-out）带外管理模块。iLO的地址在POST界面左下角有显示，

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/ilo_address.jpg)

但是这个地址跟我的网络不在一个网段啊？没办法，拎出来一台带网口的笔记本电脑，把电脑跟iLO直连，给电脑配置静态IP为`10.132.190.34`，子网掩码`255.0.0.0`，试一试，果然能ping通，问题解决。开玩笑，哥们可是网络工程专业的。

接下来就要重置iLO，省得因为上任机主的配置引出什么别的麻烦。但是问题又出现了，不知道什么原因，iLO的网页不是打不开，就是报错`Connection with iLO cannot be established`。一顿上网冲浪之后，发现还可以用SSH连接，果断掏出`putty`，好在上任没改iLO的默认管理员密码，不然我就真的抓瞎了。

登进去之后，反手就是一个`Reset /map1`来重置iLO的所有配置，过了一会iLO重启完毕，再次SSH登录，把DHCP打开，这样我就能给iLO分配一个我的内网的地址了。

```plaintext
</>hpiLO-> cd /map1/dhcpendpt1

status=0
status_tag=COMMAND COMPLETED
Tue Dec 20 15:13:47 2022



/map1/dhcpendpt1


</map1/dhcpendpt1>hpiLO-> set EnabledState=yes

status=0
status_tag=COMMAND COMPLETED
Tue Dec 20 15:14:28 2022

Network settings change applied.


Settings change applied, iLO 4 will now be reset.
Logged Out: It may take several minutes before you can log back in.

CLI session stopped
```

然后iLO会自动重启，过几分钟重启完成之后，就能在路由器里面看到iLO分配的地址了。

## 更新iLO固件

上面一顿操作重置iLO之后，很神奇地，iLO的web端能正常打开了。那么接下来，我就要升级iLO固件的版本，因为自带的iLO版本太低，反正自己用着玩，用新不用旧。

到HPE Integrated Lights-Out 4 (iLO 4)这里进入`Online ROM Flash Component for Windows x64 - HPE Integrated Lights-Out 4`下载最新的iLO 4固件。目前最新版是`2.81`。下载好安装程序之后，打开运行，选择`Extract`把安装包解压缩到一个地方。`Install`是不能用的，因为只能在服务器上才能自动安装。解压后可以看到一个叫`ilo4_281.bin`的文件，这个就是iLO的固件。

进入iLO控制台的`Administration -> Firmware`页面，`File`里面选择`ilo4_281.bin`这个文件。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/upoad_ilo4_firmware.jpg)

然后点`Upload`，静等固件上传和更新完成即可。期间不要刷新页面或者关掉页面内的进度条弹窗，地球人都知道，刷机期间手贱就是找事。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/upgrading_ilo4_firmware.jpg)

## 修复iLO降级

iLO重启之后，我看到一条报错说`iLO Self-Test reports a problem with: Embedded Flash/SD-CARD`，在网上一顿冲浪之后，说可以尝试在`Diagnostics`页面里Reset iLO来修复。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/fixing-embedded-flash-error.png)

但是我在Reset之后，问题依旧存在，按照网上的说法来看，应该是硬件有问题，要更换板载Flash了。

更新：发现了另一篇博客\[Fix\] ILO HP Error – Embedded Flash/SD-CARD: Embedded media manager failed initialization，说强制格式化NAND可以修复，我试了下，格式化并重启之后，一开始虽然显示正常，但几分钟后依旧会报同样错误。

此外根据HPE官方的advisory：

> Step 1) Upgrade the iLO 4 firmware to version 2.61  
> Step 2) Perform a NAND format  
> Step 3) Check the iLO status If the iLO status is normal, then skip to Step 6 If the iLO status is still degraded, continue to Step 4  
> Step 4) Schedule downtime; AC power-cycle and repeat the NAND format  
> Step 5) Check the iLO status If the iLO status is normal, continue to  
> Step 6 If the iLO status is still degraded, then skip to Step 7  
> Step 6) Perform these final steps if the system board does not need to be replaced: Reboot the server; reinstall IP; and refresh the server in OneView (if server is managed by OneView)  
> Step 7) If steps 1-4 did not resolve the degraded iLO, replace the system board.

确实走到了`Step 7`，只能`replace the system board`。╮(╯▽╰)╭

## 配置RAID

> 这一部分因为不方便复现，故使用HP MicroServer Gen8不疼使用教程 - Eddie’s Blog的文章配图用作参考。向Eddie表示由衷的感谢！

我在3和4盘位插了两块1TB的固态硬盘，1和2盘位暂时空下来。这么做的原因是，1和2盘位是SATA-3，3和4盘位是SATA-2，所以我打算把更快的接口留给日后拓展更大容量的机械硬盘，而固态硬盘本身读写速度就很快，插在一个慢点的接口上也不会造成很大的性能问题。

插硬盘之前要先关机，因为Gen 8的硬盘位是不支持热插拔的。插好硬盘后开机，在POST界面出现`HP Dynamic Smart Array B120i`的初始化信息时敲F5来进入RAID配置页面。这里可以多敲几次来保证成功率，就像进BIOS敲del那样。如果进入成功，就会看到一个写着HP的大球，这时候系统正在加载RAID卡的配置界面。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/smart-storage-administrator-loading.png)

进来之后，就可以点`Dynamic Smart Array B120i RAID`页面的`Configure`按钮来配置RAID了。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/configure-raid.png)

## 装ESXi

前面洋洋洒洒整了一大堆，但那都是前置任务。到这一步开始主线，装系统。Gen 8主板上带了一个TF卡槽和一个USB 2.0接口。起初我打算把ESXi装在TF卡里面的，但是试了一张之前用过的老卡，安装慢的令人发指，又试了一张几个月前买的卡，结果ESXi安装程序发现不了，怀疑是系统没识别。我懒得折腾卡了，遂拿出一个闲置的U盘，插在了主板的USB口上。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/usb_stick_inside.JPEG)

至于ESXi系统，我选择了惠普定制的版本，毕竟不知道VMWare官方版跟定制版差别在哪，还不如直接选厂商定制的版本来的方便。镜像可以到这里下载，记得要选`Pre-Gen9 Custom Image`。下载好Install CD之后，进入iLO的远程控制台，HTML5的就行，然后点击标题栏的光盘按钮，选`CD/DVD - Local *.iso file`，就可以把ESXi的安装盘插进服务器的虚拟光驱。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/ilo_remote_dvd.png)

打开服务器电源之后，就会正常从光盘启动进入ESXi的安装程序。安装步骤很简单，这里就不再赘述了，唯一要注意的就是选对安装位置，因为安装程序会把一整块盘当作系统盘，所以我个人建议把它装在TF卡或者U盘里。也不用担心TF卡或者U盘会降低系统性能，我这里实测没有什么明显难忍的性能问题。

## 换阵列卡驱动，更新系统

> 这里参考了第三篇：虚拟化层 - MicroServer Gen8 + ESXi Server 6.5u3 部署踩坑记 - 天一先生的游牧生活，这里向作者表示感谢！

装好ESXi之后，我先装上了`Ubuntu Server 22.10`和`Windows Server 2019 Datacenter`。用了一段时间，总感觉有点奇怪，尽管说这是好多年前的机器，性能不强我是有心理准备的，但也不至于点个开始菜单都要好几秒，打开资源管理器也要将近半分钟。今天等待程序载入的时候看了眼任务管理器，发现磁盘占用率竟然100%，要知道我这是两块固态硬盘组的阵列诶！既然有问题，就上网搜一搜呗，这不冲浪不知道，一冲浪吓一跳，惠普给`ESXi 6.x`打包的`B120i`板载阵列卡驱动有性能问题，会导致磁盘读写异常缓慢。针对这个问题，网上最普遍的一个解决方案就是把阵列卡驱动退回到`ESXi 5.x`中的最后一个版本`5.5.0-88`。

那话不多说，搞它！

首先，把所有虚拟机关机，全部关机后，在ESXi的`主机`页面里，点击`操作`菜单中的`进入维护模式`；在点击`操作`菜单，进入`服务`，选择`启用 Secure Shell(SSH)`，因为后面要SSH进ESXi里面安装驱动。

下载`http://downloads.hpe.com/pub/softlib2/software1/pubsw-linux/p1749737361/v98767/hpvsa-5.5.0-88.zip`，完成后，解压这个ZIP包，得到驱动本体`scsi-hpvsa-5.5.0-88OEM.550.0.0.1331820.x86_64.vib`。将这个`vib`文件上传到ESXi的`/var/log/vmware`目录下。上传成功后，SSH进入ESXi，执行如下命令：

```bash
# 卸载当前版本的驱动
esxcli software vib remove -n scsi-hpvsa -f

# 安装5.5.0-88版驱动
esxcli software vib install -v file:scsi-hpvsa-5.5.0-88OEM.550.0.0.1331820.x86_64.vib --force --no-sig-check --maintenance-mode

# 重启ESXi
reboot
```

重启完了，进ESXi看了眼驱动，发现RAID也没认出来，驱动也变成`vmw_achi`了，这是没认出来阵列卡驱动啊。搜了一下，发现可以强制禁用`vmw_ahci`，于是再进SSH，执行`esxcli system module set --enabled=false --module=vmw_ahci`这条命令，再重启之后，RAID回来了，驱动也变成`hpvsa`了。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/confirm-b120i-driver.png)

如果你在禁用`vmw_ahci`之后还不能成功加载`hpvsa`驱动，那么就得先执行`esxcli software vib remove -n scsi-hpvsa -f`来卸掉它，重启之后再重装。这时候千万不能操作硬盘，否则可能造成数据损毁。

换驱动的时候了解到，VMWare也会给ESXi推出一些系统修正补丁，那有补丁当然要打啊。一开始我想着参照ESXi 6.5 - VMware ESXi Patch Tracker这个网站给的命令在线更新，但是一直报`Got no data from process`这个错误，VMWare Communities上面也有人遇到同样的错误，解决方法是下载补丁然后手动安装。

> 关于`Got no data from process`这个错误，根据How to Upgrade from VMware vSphere ESXi 6.7 to 7.0中的描述：
> 
> > If you see the error message:
> > 
> > Got no data from process: LANG=en\_US.UTF-8 /usr/lib/vmware/esxcli-software profile.update -d “https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml“ -p “ESXi-7.0.0-15843807-standard”,
> > 
> > check whether there is a persistent scratch partition. There may not be a persistent scratch partition if you’re running ESXi from a USB flash drive.
> 
> 而根据How to Run ESXi from a USB Flash Drive所说：
> 
> > ![Installing ESXi on an 8-GB drive (or bigger)](https://blog-static.boris1993.com/setting-up-microserver-gen8/esxi-partition-map-hard-drive.webp)  
> > If ESXi is installed on a USB flash drive or SD card whose size is 8 GB or more, partitions #2 and #3 are not created. The Scratch directory used to store logs is linked to the /tmp/scratch directory that is located on the RAM drive.
> 
> 也就是说，如果把ESXi装在大于等于8GB的TF卡或者U盘里面的话，就会缺少一个`scratch`分区。
> 
> 另外根据这个分区图，我还发现，如果把ESXi装在硬盘里的话，它会把剩余空间作为数据存储，而不是像网上一些文章说的只用来装系统，这个我后面有时间会再测试一下。

那么，首先到Product Patches这里，Product选`ESXi (Embeeded and Installable)`，Version选`6.5.0`（如果你装的不一样，以你为准），点Search后会列出一堆补丁，选最新那个，点Download Now来下载补丁的ZIP包。然后点`View Details`，我们要在详细信息页面找到Image Profile的名字。

![](https://blog-static.boris1993.com/setting-up-microserver-gen8/vmware-product-patches.png)

> 关于什么是Image Profile：  
> VMWare文档说：Image profiles define the set of VIBs that an ESXi installation or update process uses.  
> 也就是说，Image Profile定义了包里的哪些VIB会被安装。  
> 每个补丁包中都会带有2~4个Image Profile，根据VMware ESXi Image Profiles的描述，各个Image Profile的含义是：
> 
> 1.  ESXi-5.5.0-20140704001-standard - 包含全部补丁
> 2.  ESXi-5.5.0-20140704001-no-tools - 包含除了VMware Tools之外的全部补丁
> 3.  ESXi-5.5.0-20140701001s-standard - 只包含安全补丁
> 4.  ESXi-5.5.0-20140701001s-no-tools - 只包含安全补丁，同时没有VMware Tools

因为我要更新所有的VIB，所以我将要用`ESXi-6.5.0-20221004001-standard`这个Image Profile。SSH进ESXi，执行`esxcli software profile update -p ESXi-6.5.0-20221004001-standard -d /vmfs/volumes/63a1b271-3340b157-711e-d0bf9c4694d8/ESXi650-202210001.zip`开始安装（记得把`-d`参数指定的绝对路径改为你的机器上面的路径）。安装过程比较耗时间，这期间可以`tail -f /var/log/esxupdate.log`来看ESXi正在干什么，毕竟看着日志在跑，心里也有底，不然总害怕系统是不是死掉了。

更新成功后，`esxcli`会打出来哪些包被更新了，但是实在是太多，我根本懒得看，直接重启。重启之后，ESXi能正常进系统，存储和虚拟机都正常载入，说明没出啥毛病。这时候就可以退出维护模式，给虚拟机开机。有一说一，现在不管是开机还是打开什么东西，反应确实快了不少，看来驱动导致的硬盘性能问题已经成功解决。

## 未完待续

![](https://blog-static.boris1993.com/to-be-continued.png)
