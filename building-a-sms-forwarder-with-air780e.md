---
title: "100块自制短信转发器"
source: https://www.boris1993.com/building-a-sms-forwarder-with-air780e.html
date: 2023-03-21
updated: 2026-10-10
tags: [LuatOS, 合宙, 短信转发, Air780E, ESP32S3]
categories: [瞎折腾]
---

# 100块自制短信转发器

迫于手上有张KnowRoaming的卡用来收短信，但是为了这玩意再照顾个手机又很烦，正好看到合宙有卖4G模块，还有人已经实现了短信转发功能。本着生命在于折腾的精神，果断下单了一张开始搞。

## 成本

-   Air780E + ESP32S3 套餐 - ￥66.80

因为KnowRoaming手机卡只能收短信，不能发短信，也没有数据流量，而Air780e正好不支持连接Wi-Fi，所以需要搭配ESP32S3。ESP32S3用来连接Wi-Fi，并作为主控板，通过UART来连接和驱动Air780e。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/air780e_purchase_history.png)

-   16pin排母 - ￥15

合宙只给送了排针，没有排母（甚至不卖这玩意），但是直接把ESP32S3和Air780e焊死显然不合适，所以只能自己再买排母。￥15一盒里面有5对，所以实际分摊下来只有￥3。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/socket_header.jpg)

-   USB Type-C转Type-B双公头转接头 - ￥9.9

开发过程中可以用线凑合，但是我最后打算直插在UPS的USB口上，所以转换头必不可少。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/usb_c_to_b_converter.png)

-   找师傅焊接的手工费 - ￥50

这没啥说的，自己又没有电烙铁又没这技术，就找了个修手机的铺子，让老板帮忙焊的。老板一边焊，一边说他这含银焊锡用在这上面都浪费……

## 焊接和组装

先上个全家福~

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/parts.jpeg)

焊排针和排座没啥好说的，就照着下图的样子焊就行。

| ESP32S3 | Air780e |
| --- | --- |
| ![](https://blog-static.boris1993.com/forwarding-sms-by-air780/esp32s3.jpeg) | ![](https://blog-static.boris1993.com/forwarding-sms-by-air780/air780e.jpeg) |

另外因为Air780e默认不会上电自动开机，所以还要把PWR键上方的焊盘短接来让它通电就自动开机。（暂且借晨旭的博客中的图片，因为我焊的时候忘了这茬了……）

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/shorting_soldering_pad_for_power_key.jpg)

然后SIM卡芯片朝板子，缺口朝内，插进Air780e的SIM卡槽。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/sim_card_direction.jpeg)

最后按照如图的方向把排针对其插上就组装好了。一定要注意方向，毕竟有5V供电的针脚，插反了把5V怼到GND上会发生什么就不好说了，或许可以参考伊朗电工Mehdi Sadaghdar的各种作死实战。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/put_together.jpeg)

## 刷机和测试

ESP32S3接上USB转换头，再插到电脑上，就可以愉快的刷机了。代码我已经写好并发布在GitHub上面了，下载下来，照着README里面刷机的部分改好代码，用合宙的`LuaTools`刷进去，就可以了。注意板子的固件要用我的repo里面的，`LuaTools`提供的固件刷进去之后可用内存不够，会发不出去HTTP请求。如果不会刷机，可以看看ESP32C3烧录教程。

刷完机板子会自动重启，这时候可以观察板子上面指示灯的状态，或者直接在`LuaTools`里面看日志，来看板子的初始化状态。LED灯各种状态的含义也可以看README里面的解释。

等板子初始化完成之后，就可以给这个手机号发短信了，板子收到短信后就会把发信号码和短信内容发送到配置的通知通道。开发的时候我已经测试过普通短信、长短信、中文、英文、日语，都可以正常接收。

![](https://blog-static.boris1993.com/forwarding-sms-by-air780/test_results.JPEG)
