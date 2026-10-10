---
title: "修复被切换成RNDIS模式后改不回来的大疆4G模块"
source: https://www.boris1993.com/dji-4g-module-recover-from-rndis-mode.html
date: 2026-08-09
updated: 2026-10-10
tags: [大疆4G模块]
categories: [瞎折腾]
---

# 修复被切换成RNDIS模式后改不回来的大疆4G模块

[上回书](https://www.boris1993.com/dji-4g-module-for-ipad-networking)说到可以把大疆4G模块变成iPad的外置网卡，帖子发出来之后，有好奇宝宝找我说，把`usbnet`改成3之后改不回来了，给`Quectel USB AT Port`发指令只会返回乱码。那这篇帖子就是解决这个问题的。

两个办法，一个是用带USB Type-C口的Android手机装Godles-lab/usb-modem-sms，用它改回来。找不到的可以到我的网盘下载。

| 点右上角三个点，打开`USB模式`菜单 | 切换USB模式到ECM |
| --- | --- |
| ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/usbmodem-sms-step-1.png) | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/usbmodem-sms-step-2.png) |

用电脑也能修，但是要稍微麻烦点。

-   首先卸载之前安装过的`Quectel_Windows_USB_DriverQ_NDIS`驱动，它会与接下来要安装的驱动冲突
-   然后安装`Quectel_Windows_USB_DriverM_RNDIS_V1.8_CN.zip`里面的驱动（找不到的可以到我的网盘）下载
-   把模块插上电脑，等模块启动完毕后，在设备管理器中能看到4个名为`Baiwang`的未知设备，找到硬件ID以`MI_05`结尾的那个  
    ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-mi_05.png)
-   在找到的那个设备上点右键，选`更新驱动`，在弹出的窗口中依次选择第二项`浏览本机` -> 窗口底部的`从列表中选择` -> 列表中的`端口 (COM & LPT)` -> 左侧栏选`dji-innovations inc.`，右侧栏选`DJI USB Virtual COM` -> 弹出的警告中选择继续
    
    | 第一步 | 第二步 | 第三步 | 第四步 | 第五步 |
    | --- | --- | --- | --- | --- |
    | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-update-driver.png) | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-browse-computer-for-driver.png) | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-pick-from-list.png) | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-pick-from-ports-drivers.png) | ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/baiwang-dji-virtual-com.png) |
    
-   打开`llcom`，选择名为`DJI USB Virtual COM`的端口并打开，确认勾选`发末尾加回车换行`（如图打了叉的状态），发送`AT`测试，如果返回OK说明前面的步骤都没错  
    ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/llcom-virtual-com.png)
-   然后发送`AT+QCFG="usbnet",1`，就可以改回ECM模式了  
    ![](https://blog-static.boris1993.com/dji-4g-module-recover-from-rndis-mode/llcom-set-usbnet.png)

后面再要改usbnet模式的话，需要再装回`Quectel_Windows_USB_DriverQ_NDIS`驱动，再用llcom发指令。或者再到Android手机上用`Godles-lab/usb-modem-sms`改。
