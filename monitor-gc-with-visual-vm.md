---
title: "使用VisualVM监控应用GC"
source: https://www.boris1993.com/monitor-gc-with-visual-vm.html
date: 2020-02-09
updated: 2026-10-10
tags: [Java, VisualVM]
categories: [学知识]
---

# 使用VisualVM监控应用GC

我们知道，使用VisualVM可以监控Java应用的各种运行时信息，包括资源占用、正在运行的线程等等。本文将简单介绍如何使用VisualVM的插件`Visual GC`来监控Java应用的垃圾回收情况。

## 安装`Visual GC`插件

因为`Visual GC`插件并不是随VisualVM附带的，需要到插件商店下载。

点击`Tools`–>`Plugins`打开插件管理窗口，进入`Available Plugins`，在左侧窗格中勾选`Visual GC`，点击`Install`，同意许可协议后，稍等片刻即可完成安装。安装完成之后无需重启VisualVM。

![Install Visual GC](https://blog-static.boris1993.com/monitor-gc-with-visualvm/install-visual-gc.png)

## 监控垃圾回收

安装完成后，在VisualVM的`Applications`窗格中双击要监控的应用，然后在右侧窗格中进入`Visual GC`，即可以图形化的方式查看当前该应用中各个内存区域的情况。

![Monitor GC](https://blog-static.boris1993.com/monitor-gc-with-visualvm/monitor-gc.png)

窗格中的`Spaces`部分展示的是当前各个区域的占用情况；`Graphs`部分则包括：

-   `Compile Time`：编译时间，包括编译的次数，和累计的编译时间。图表中的一次脉冲就代表一次编译，脉冲越宽代表编译时间越长
-   `Class Loader Time`：类加载时间，包括加载了多少个类(`loaded`)，卸载了多少个类(`unloaded`)，和累计用于类加载的时间
-   `GC Time`：垃圾回收时间，包括垃圾回收的次数，垃圾回收消耗的时间，和上一次垃圾回收的原因
-   `Eden Space`：Eden区的空间，括号中第一位是最大容量，第二位是当前容量，其后冒号跟着的是当前使用了的大小，再往后是发生垃圾回收的次数，和垃圾回收消耗的时间
-   `Survivor 0`和`Survivor 1`：两个`Survivor`区的空间，同样包括最大容量、当前容量，和当前使用了的大小
-   `Old Gen`：老年代的空间，同样包括最大容量、当前容量、当前使用了的大小、发生垃圾回收的次数，和垃圾回收消耗的时间
-   `Metaspace`：Metaspace的空间(如果使用的是Java 1.7及之前版本，这里则是`Perm Gen`)，包括最大容量、当前容量，和当前使用了的大小
