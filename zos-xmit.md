---
title: "使用TRANSMIT和RECEIVE命令打包、解包文件"
source: https://www.boris1993.com/zos-xmit.html
date: 2017-11-19
updated: 2026-10-10
tags: [Mainframe, z/OS, TRANSMIT, XMIT, RECEIVE]
categories: [学知识]
---

# 使用TRANSMIT和RECEIVE命令打包、解包文件

当需要从z/OS中下载一个PS文件，或者下载一个Member时，我们可以简单地使用FTP或者IND$FILE将文件下载回来，但是如果想要下载一整个PDS呢？总不能一个个地去下载吧，此时，XMIT命令就派上用场了。

* * *

# TRANSMIT命令

TRANSMIT命令用于将指定文件打包成XMIT档案以方便传输。

## 语法

使用一条命令前，必须要知道这条命令的语法。那么XMIT命令的语法如下：  
*注：TRANSMIT命令可以简写为XMIT*

```plaintext
XMIT (nodeid.username) DSNAME('input.dataset.name') [MEMBERS(member1, member2, ...)] OUTDSN('output.dataset.name')
```

其中：

-   (nodeid.username) 为目标机器的JES2 Node名，以及接收用户的TSOID，不过亲测在这里写自己的Node名和TSOID也能正常使用。通常来说，本机的Node name为N1
-   DSNAME 为要打包的数据集名字
-   如果只打包这个PDS中的一部分member，则可以在MEMBERS参数中指定。因为目前还没有用过，所以不知道可不可以指定通配符
-   OUTDSN 为打包之后输出文件的数据集名字

TRANSMIT命令的官方手册在 [**这里**](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.ikjc500/transmi.htm) ，完整的TRANSMIT命令的语法在 [**这里**](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.ikjc500/transsyn.htm) ，参数的详细说明在 [**这里**](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.ikjc500/transmitcomop.htm) 。

另外，根据IBM员工 Isabel Arnold 的建议，在打包之前最好先创建一个 `DSORG=FB,LRECL=80,BLKSIZE=3120` 的文件供TRANSMIT用作输出文件。

## 示例

如果我想要打包 IBMUSER.COBOL.SRC 这个PDS中的所有member，打包输出文件名为 IBMUSER.COBOL.SRC.XMIT 那么命令可以这样写：

```plaintext
XMIT (N1.IBMUSER) DSN('IBMUSER.COBOL.SRC') OUTDSN('IBMUSER.COBOL.SRC.XMIT')
```

* * *

# RECEIVE命令

RECEIVE命令用于解包XMIT档案。

## 语法

同样，这里先展示RECEIVE命令的语法：

```plaintext
RECEIVE INDSN('xmit.dataset.name')
```

其中 `INDSN` 为XMIT档案的文件名。

## 示例

如果现在我在另一台主机上接收到了这个XMIT档案，那么我可以使用如下命令解包这个文件：

```plaintext
RECEIVE INDSN('IBMUSER.COBOL.SRC.XMIT')
```

在RECEIVE命令成功识别指定的XMIT档案之后，会输出如下信息：

```plaintext
INMR901I Dataset IBMUSER.COBOL.SRC from IBMUSER on NODENAME
INMR906A Enter restore parameters or 'DELETE' or 'END' +
```

此时RECEIVE命令等待用户输入解包信息，我们可以回复如下命令：

```plaintext
DA('IBMUSER.COBOL.SRC')
```

来将内容解包至 IBMUSER.COBOL.SRC 中。

此处需要注意的是，如果目标数据集不存在，则RECEIVE会自动创建一个同名数据集，但接下来的解包过程可能会因这个数据集的空间不够用于存放解包出来的文件而报出 `ABEND B37` 。为避免这种情况发生，建议在RECEIVE前预先创建好需要的数据集，并保证数据集的空间足够。

在输入DA命令后，RECEIVE将会试图向指定位置解包，并且会将结果输出至终端。

## 对INMR906A消息的回复

对 `INMR906A` 的消息，有如下三种回复：

-   DATASET(‘output.dataset.name’) - 将XMIT档案解包至指定位置，可简写为DA()
-   DELETE - 删除该XMIT档案
-   END - 退出，不执行任何操作

# 参考文档

-   TRANSMIT command
-   TRANSMIT command syntax
-   TRANSMIT command operands
-   RECEIVE command
-   Receiving Data Sets with the RECEIVE Command
-   Transfering Load Modules between Mainframes using XMIT and ftp
