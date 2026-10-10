---
title: "让iPad用一代大疆4G模块上网"
source: https://www.boris1993.com/dji-4g-module-for-ipad-networking.html
date: 2026-07-27
updated: 2026-10-10
tags: [大疆4G模块]
categories: [瞎折腾]
---

# 让iPad用一代大疆4G模块上网

跟风买了个大疆4G模块，给iPad当外置网卡用，省得次次还得拿手机开热点。

话说这算不算花小钱办大事，毕竟蜂窝版的要比WiFi版的贵一千多块呢，这70块就搞定了。

操作非常简单，只需要下面几步：

-   安装驱动和必要工具
    -   下载移远的NDIS驱动，可以从官网下载（需要先注册账号），或者去我的网盘备份里下载
    -   llcom，官方GitHub Releases在这里，当然我的网盘备份里也有
-   到设备管理器中检查，如果驱动安装成功，那么在把模块接上电脑后，可以在设备管理器中看到`Quectel USB AT Port (COM6)`这个串口设备

![](https://blog-static.boris1993.com/dji-4g-module-for-ipad-networking/device-manager-quectel-at-port.png)

-   打开`llcom`，在下拉菜单中选择`Quectel USB AT Port (COM6)`这个串口，点击`打开串口`
    -   在底部文本框输入`AT`，点击`发送`，如果上面的日志中看到模块返回`OK`即说明驱动和连接都正常
    -   清空文本框内容，输入`AT+QCFG="usbnet",1`，点击发送，接下来模块会自动重启，大约耗时半分钟
    -   在模块重启完成后再次连接相同串口，清空文本框并输入`AT+QCFG="usbnet"?`，如果模块返回`+QCFG: "usbnet",1`，就说明修改成功了

![](https://blog-static.boris1993.com/dji-4g-module-for-ipad-networking/llcom.png)

然后就可以把模块从电脑上拔下来，用USB C-to-C线连上iPad，如果在设置里能看到`以太网`，点进去有个叫`Baiwang`的设备，并且用浏览器什么的测试可以成功上网，就结束了。

哦对，别忘了插SIM卡。
