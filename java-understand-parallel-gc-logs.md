---
title: "看懂ParallelGC的日志"
source: https://www.boris1993.com/java-understand-parallel-gc-logs.html
date: 2020-02-05
updated: 2026-10-10
tags: [Java, 垃圾回收, GC, JVM]
categories: [学知识]
---

# 看懂ParallelGC的日志

在排查垃圾回收相关的问题时，一个必不可少的技能就是要能看懂Java的垃圾回收日志。本文将介绍打印GC日志相关的JVM参数，以及使用不同参数时JVM将会打印出的日志内容。

## JDK版本

```text
openjdk version "1.8.0_232"
OpenJDK Runtime Environment (AdoptOpenJDK)(build 1.8.0_232-b09)
OpenJDK 64-Bit Server VM (AdoptOpenJDK)(build 25.232-b09, mixed mode)
```

## 示例代码

在演示过程中，我们将通过在死循环中构造大对象的方式来触发垃圾回收

```java
public class GCExample {
    public static void main(String[] args) {
        String[] strings;
        while (true) {
            strings = new String[1024];
        }
    }
}
```

同时，为了让垃圾回收更早的开始，我们还需要指定一个较小的堆内存，比如我选择只分配10MB的空间。

```text
-Xmx10m
-Xms10m
```

## GC日志相关的JVM参数

与GC日志相关的JVM参数有如下8个：

| 参数名 | 参数含义 |
| --- | --- |
| \-XX:+PrintGC | 输出简要GC日志 |
| \-verbose:gc | 等同于 -XX:+PrintGC |
| \-XX:+PrintGCDetails | 输出详细GC日志 |
| \-Xloggc:gc.log | 输出GC日志到文件gc.log |
| \-XX:+PrintGCTimeStamps | 输出GC的时间戳（以JVM启动到当期的总时长的时间戳形式） |
| \-XX:+PrintGCDateStamps | 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800） |
| \-XX:+PrintHeapAtGC | 在进行GC的前后打印出堆的信息 |
| \-XX:+PrintReferenceGC | 打印年轻代各个引用的数量以及时长 |

### `-XX:+PrintGC`与`-verbose:gc`

虽然上面说这两个参数是等价的，但是，`-verbose:gc`是一个标准的参数，而`-XX:+PrintGC`则在JDK 9之后被标为废弃的(deprecated)，所以建议使用`-verbose:gc`代替`-XX:+PrintGC`。

## 打开`-verbose:gc`

使用这个参数将会打开简略的GC日志。打开这个参数并运行程序，很快就可以看到这样的GC日志：

```plaintext
[GC (Allocation Failure)  2520K->472K(9728K), 0.0005722 secs]
```

我们将上面的日志拆分开来：

-   `GC`代表这次垃圾回收的类型。`GC`代表发生了`Minor GC`，`Full GC`代表发生了`Full GC`
-   `Allocation Failure`简述了发生GC的原因。在这个例子中，是因为在年轻代中没有合适的空间导致新的对象分配失败，从而发生GC
-   `2520K->472K`说明了年轻代的占用量变化。在这个例子中，年轻代的使用量从2520KB减少到了472KB
-   `9728K`表示整个堆的大小
-   `0.0005722 secs`表示了本次GC消耗的时间

## 打开`-XX:+PrintGCDetails`

使用这个参数后，将会打印详细的GC日志，它打印的内容包含了`-verbose:gc`参数打印的内容。

```plaintext
[GC (Allocation Failure) [PSYoungGen: 2048K->0K(2560K)] 2500K->452K(9728K), 0.0003785 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

这个参数打印的日志与上一个参数打印的日志有一部分重合，这里我们关注新增的内容：

-   `[PSYoungGen: 2048K->0K(2560K)]`表明了使用的垃圾回收器、年轻代的占用量变化，和年轻代的大小
-   `2500K->452K(9728K)`表明了整个堆的占用量变化，和整个堆的大小
-   `[Times: user=0.00 sys=0.00, real=0.00 secs]`表示本次GC消耗的时间
    -   `user`代表垃圾回收器消耗的CPU时间
    -   `sys`代表系统调用或等待系统事件消耗的时间
    -   `real`代表应用实际停止的时间，近似于`(user + sys) / 垃圾回收器使用的线程数`

## 打开`-XX:+PrintGCTimeStamps`

这个参数不影响日志的详细程度，在打开之后，日志中会出现从JVM启动到出现日志时的毫秒数，比如下面例子的`1.523`。

```plaintext
1.523: [GC (Allocation Failure) [PSYoungGen: 2141K->96K(2560K)] 4805K->2776K(9728K), 0.0004359 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

## 打开`-XX:+PrintGCDateStamps`

这个参数也不影响日志的详细程度，在打开之后，日志头部会加上打印日志的时间和时区，比如下面例子的`2020-02-06T10:13:08.634-0800`。

```plaintext
2020-02-06T10:13:08.634-0800: 1.122: [GC (Allocation Failure) [PSYoungGen: 2048K->0K(2560K)] 3328K->1280K(9728K), 0.0004243 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

## 打开`-XX:+PrintHeapAtGC`

使用这个参数后，在打印GC日志时，会同时打印出堆在GC前后的信息，即根据不同的内存区域，分别显示在垃圾回收前后的空间使用情况。同时，日志中还以`[bottom,top,end)`的格式展现出了各个内存区域的地址，其中`bottom`为这个区域的起始位置，`top`为这个区域当前的顶部位置，`end`为这个区域可使用的地址上限\[^1\]。

此外，日志中关于`Metaspace`相关的信息可以参考注脚4\[^4\]指向的那篇回答。

```plaintext
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 2560K, used 2047K [0x00000007bfd00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 2048K, 99% used [0x00000007bfd00000,0x00000007bfeffe78,0x00000007bff00000)
  from space 512K, 0% used [0x00000007bff80000,0x00000007bff80000,0x00000007c0000000)
  to   space 512K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007bff80000)
 ParOldGen       total 7168K, used 0K [0x00000007bf600000, 0x00000007bfd00000, 0x00000007bfd00000)
  object space 7168K, 0% used [0x00000007bf600000,0x00000007bf600000,0x00000007bfd00000)
 Metaspace       used 3045K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 320K, capacity 388K, committed 512K, reserved 1048576K
2020-02-06T10:21:05.317-0800: 0.219: [GC (Allocation Failure) [PSYoungGen: 2047K->496K(2560K)] 2047K->512K(9728K), 0.0019061 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 2560K, used 496K [0x00000007bfd00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 2048K, 0% used [0x00000007bfd00000,0x00000007bfd00000,0x00000007bff00000)
  from space 512K, 96% used [0x00000007bff00000,0x00000007bff7c010,0x00000007bff80000)
  to   space 512K, 0% used [0x00000007bff80000,0x00000007bff80000,0x00000007c0000000)
 ParOldGen       total 7168K, used 16K [0x00000007bf600000, 0x00000007bfd00000, 0x00000007bfd00000)
  object space 7168K, 0% used [0x00000007bf600000,0x00000007bf604000,0x00000007bfd00000)
 Metaspace       used 3045K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 320K, capacity 388K, committed 512K, reserved 1048576K
}
```

## 打开`-XX:+PrintReferenceGC`

打开这个参数后，就可以看到各种引用的数量，以及对其垃圾回收所消耗的时长。

```plaintext
[GC (Allocation Failure) [SoftReference, 0 refs, 0.0000125 secs][WeakReference, 0 refs, 0.0000054 secs][FinalReference, 0 refs, 0.0000045 secs][PhantomReference, 0 refs, 0 refs, 0.0000222 secs][JNI Weak Reference, 0.0000037 secs][PSYoungGen: 2112K->64K(2560K)] 3488K->1456K(9728K), 0.0005389 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

\[^1\]: Meaning of heap memory addresses in GC logs using -XX:+PrintHeapAtGC?  
\[^2\]: Parallel GC - Plumber  
\[^3\]: Parallel Full GC - Plumber  
\[^4\]: Understanding metaspace line in JVM heap printout
