---
title: "简单理解NTP v3协议"
source: https://www.boris1993.com/understanding-ntp-protocol.html
date: 2022-09-07
updated: 2026-10-10
tags: [NTP]
categories: [学知识]
---

# 简单理解NTP v3协议

前段时间瞎折腾，给自己的黑莓Bold 9900写了个通过NTP同步时间的小工具，顺便在这里记录一下我在实现一个NTP客户端时对这个协议的理解。

## 端口号

NTP协议使用UDP作为传输层协议，服务器监听UDP端口123，在收到有效的报文后，服务器会发送响应报文，否则服务器将直接忽略不做响应。

## 时间格式

NTP协议使用三种时间格式。

### NTP短时间格式

短时间格式长度为32位，其中高16位代表从NTP时间戳0秒至现在的秒数，低16位代表1秒以内的分数部分。  
这个格式只会在NTP报文的delay和dispersion字段中用到。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Seconds              |           Fraction            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### NTP时间戳

NTP时间戳格式长度为64位，其中高32位代表从NTP时间戳0秒至现在的秒数，低32位代表1秒以内的分数部分。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Seconds                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Fraction                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### NTP日期格式

NTP日期格式长度为128位，其中高32位用来表示NTP时间纪元，然后用32位表示从当前纪元开始经过的秒数，最后用64位表示1秒以内的分数部分。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Era Number                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Era Offset                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                           Fraction                            |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## 报文格式

一个NTP v3的报文必须包含如下字段：

-   `LI` - Leap Indicator，2 bit整型数，指示当月最后一分钟是否包含闰秒
-   `VN` - Version Number，3 bit整型数，指示NTP协议的版本号。如NTP v3就是3。
-   `MODE` - 3 bit整型数，指示发包方的工作模式。通常来说客户端使用3(client)请求时间，服务端使用4(server)返回时间。
-   `STRATUM` - 8 bit整型数，代表NTP层数。0代表时钟源，如装备有GPS接收机的主服务器；1-15逐层作为下游服务器，16被定义为“无法同步”。
-   `POLL` - 8 bit有符号整型数，代表在间隔多少秒后再进行下一次同步。值由`log2(second)`计算得出。
-   `PRECISION` - 8 bit有符号整型数，代表系统时钟的精确度。
-   `ROOT DELAY` - NTP短时间格式，指示从客户端到根服务器(stratum 1的服务器)的延迟。
-   `ROOT DISPERSION` - NTP短时间格式，指示数据从根服务器到客户端之间可能引入的误差。
-   `REFERENCE ID` - 32 bit代码，用于标识一个特定的服务器，或一个参考时钟。
    -   对于stratum 0的数据包，该字段为4个ASCII字符，称作“kiss code”，用于调试和监控。
    -   对于stratum 1的数据包，该字段为参考时钟的标识符。标识符由IANA维护，此外以“X”开头的标识符都被预留给未注册的试验和开发用途。
    -   对于stratum 2~15的数据包，该字段为服务器的标识符。当服务器使用IPv4时，该字段为服务器的IP地址；当服务器使用IPv6时，该字段为IPv6地址的前四段。
-   `REFERENCE TIMESTAMP` - NTP时间戳格式，内容为客户端最后同步的时间。
-   `ORIGIN TIMESTAMP` - NTP时间格式，内容为数据包离开客户端的时间。
-   `RECEIVE TIMESTAMP` - NTP时间格式，内容为数据包抵达服务器的时间。
-   `TRANSMIT TIMESTAMP` - NTP时间格式，内容为数据包离开服务器的时间。
-   `DESTINATION TIMESTAMP` - NTP时间格式，内容为数据包抵达客户端的时间。
    -   注：DESTINATION TIMESTAMP并不会包含在数据包中，而是在客户端收到数据包之后，它的数值才会被确定。

那么全部组合起来，就是这个样子的：

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|LI | VN  |Mode |    Stratum     |     Poll      |  Precision   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Root Delay                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Root Dispersion                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Reference ID                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     Reference Timestamp (64)                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Origin Timestamp (64)                    +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Receive Timestamp (64)                   +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Transmit Timestamp (64)                  +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

然而上述字段并不需要全部填写数据，实际上除了LI、VN、MODE、STRATUM之外，剩下的所有字段都可以填零。如下就是一个我用来测试的数据包：

```text
HEX:
DB 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

拆开来看的话：

```text
BIN:
LI = 0b11 = 3 unknown (clock unsyncronized)
VN = 0b011 = 3
MODE = 0b011 = 3 client
STRATUM = 0b00010000 = 16
POLL = 0b00000000 = 0
PRECISION = 0b00000000 = 0
ROOT DELAY = 0b00000000000000000000000000000000
ROOT DISPERSION = 0b00000000000000000000000000000000
REFERENCE ID = 00000000000000000000000000000000
REFERENCE TIMESTAMP = 00000000000000000000000000000000 00000000000000000000000000000000
ORIGIN TIMESTAMP = 00000000000000000000000000000000 00000000000000000000000000000000
RECEIVE TIMESTAMP = 00000000000000000000000000000000 00000000000000000000000000000000
TRANSMIT TIMESTAMP = 00000000000000000000000000000000 00000000000000000000000000000000
```

## 计算second和fraction

计算second很简单，取出timestamp的高32位就可以了；但是从fraction计算毫秒数比较麻烦，需要通过`fraction * 10^6 / 2^32`计算得到毫秒数。

这里我给出一个Java的代码片段：

```java
final long seconds = (ntpTimestamp >>> 32) & 0xFFFFFFFFL;
final long secondsInMilliseconds = seconds * 1000;

final long fractionInTimestamp = (ntpTimestamp & 0xFFFFFFFFL);
final long milliseconds = fractionInTimestamp * Math.pow(10, 6) / Math.pow(2, 32);
```

然后计算1900年1月1日 00:00:00的UNIX时间戳作为基准UNIX时间戳，再加上`secondsInMilliseconds`和`milliseconds`，就可以得到NTP返回的当前时间了。

## 参考文档

-   Network Time Protocol Version 4: Protocol and Algorithms Specification - RFC
-   Network Time Protocol (NTP) 网络时间协定 - Jan Ho 的网络世界
-   The Root of All Timing: Understanding root delay and root dispersion in NTP
-   NTP Timestamp - Thompson’s Technological Insight
-   A Very Short Introduction to NTP Timestamps
-   NtpPacketUtils#getNtpTimestampMilliseconds - blackberry\_time\_sync\_ntp - GitHub
