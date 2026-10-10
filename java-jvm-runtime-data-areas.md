---
title: "JVM运行时的数据区域"
source: https://www.boris1993.com/java-jvm-runtime-data-areas.html
date: 2019-03-27
updated: 2026-10-10
tags: [Java, JVM, 内存模型]
categories: [学知识]
---

# JVM运行时的数据区域

JVM在运行时，会按照程序执行的需要来创建一系列的运行时数据区域。有的区域只会随JVM起停而被创建和销毁，有的区域则会独立分配给各个线程，并随线程的起停而创建和销毁。这些运行时区域，按照功能和性质不同，会分成如下几部分：

-   线程专有
    -   [pc(program counter)寄存器](#pc-program-counter-%E5%AF%84%E5%AD%98%E5%99%A8)
    -   [JVM栈](#JVM%E6%A0%88)
    -   [本地方法栈](#%E6%9C%AC%E5%9C%B0%E6%96%B9%E6%B3%95%E6%A0%88)
-   线程间共享
    -   [堆](#%E5%A0%86)
    -   [方法区](#%E6%96%B9%E6%B3%95%E5%8C%BA)
        -   [运行时常量池](#%E8%BF%90%E8%A1%8C%E6%97%B6%E5%B8%B8%E9%87%8F%E6%B1%A0)

## pc(program counter)寄存器

JVM允许同时运行多个线程，每个线程都有它自己的PC寄存器。在任意时刻，每个JVM线程都在执行一个方法中的某条语句，而这个正在被执行的方法，就叫做这个线程的“当前方法”。

如果当前方法不是一个本地(native)方法，那么PC寄存器的内容是当前正在执行的指令的地址；如果当前方法是本地方法，那么PC寄存器的值则是空(undefined)的。

## JVM栈

每个JVM都会在其启动时创建自己私有的JVM栈，栈之中存储的是[栈帧](https://www.boris1993.com/projects/java/fundamentals/java-jvm-stack-frame.html)，用于存储局部变量和方法调用信息。

规范中允许栈的深度可以是固定的，也可以根据要求动态的扩展和收缩。如果是固定深度的栈，那么每个栈的深度会在其创建时按照需要独立指定。

当请求创建的栈大于所允许的深度，那么JVM会抛出`StackOverflowError`异常；当程序试图扩大一个可以动态伸缩的栈，或者试图为新的线程创建一个栈，但是可用内存不足以完成这个操作时，那么JVM会抛出`OutOfMemoryError`异常。

## 本地方法栈

本地方法栈与JVM栈类似，保存了本地方法的调用信息。

本地方法栈的空间可以是固定的，也可以是动态伸缩的。

当程序申请了大于所允许的本地方法栈空间，那么JVM会抛出`StackOverflowError`异常；如果程序申请扩展一个可以动态伸缩的本地方法栈，或者试图创建一个栈，但是可用内存不足以满足要求时，JVM会抛出`OutOfMemoryError`异常。

## 堆

在JVM启动时，会创建一个共享于所有线程的堆空间，其中存放着所有的对象，和被分配好空间的数组。用于存放对象的空间由一个自动化的存储空间管理机制，即垃圾回收机制(garbage collector)，来进行管理。堆空间可以是固定大小的，也可以是按需伸缩的。

如果程序试图申请扩大堆空间，但是存储管理机制无法满足需求时，JVM会抛出`OutOfMemory`异常。

在堆中，JVM又根据作用不同，将内存空间分为如下几部分：

### 新生代(New generation)

新生代保留的是生命周期短，并且很快就会被回收掉的对象。其中的空间又随着“复制算法”这一垃圾回收算法而被分为`Eden Space`和`Survivor Space`。具体可以参考[Java的垃圾回收算法](https://www.boris1993.com/projects/java/fundamentals/java-gc-algorithms.html)这篇博文。

### 老年代(Tenured generation)

在多次垃圾回收后仍然存活的对象，将会被放到老年代空间中。因此可以认为，老年代中的对象的生命周期都是比较长的。

## 方法区

方法区(method area)是一个共享于所有JVM线程的空间，创建于JVM启动时，其中主要存放的是类的元数据，包括类的类型信息、常量池、方法数据、方法的代码等，这些数据主要来源于class文件。方法区逻辑上属于堆的一部分，但是为了与堆区分开来，方法区通常又叫`非堆`。

类型信息包括类的完整名称、父类的完整名称、类型修饰符(`private`/`protected`/`public`)，和类型的直接接口类表。

方法的数据包括方法的名称、返回类型、参数、方法的修饰符、字节码、操作数栈和方法栈帧的局部变量区大小，和异常表。

方法区的大小可以是固定的，也可以是按需伸缩的，但是根据虚拟机实现的不同，垃圾回收机制可能不会回收或压缩方法区的空间。

如果方法区的可用内存无法满足一次申请空间的请求，那么JVM会抛出`OutOfMemoryError`异常。

### 永久代和Metaspace

在HotSpot VM中，永久代和Metaspace就是方法区的具体实现。在Java 8之前，方法区是以永久代的形式存在的；而从Java 8之后，永久代就被Metaspace取而代之了。

在Java 1.7和之前版本中，永久代是一块独立于堆的内存空间，在物理内存上与堆是连续的。同时，在Java 1.7中，一部分原属于永久代的内容也在逐步被移动到其他位置，比如符号引用被移动到了本地内存(native memory)中，字符串常量池和类的静态变量则被移动到了堆中。

从Java 8开始，永久代被Metaspace取而代之。Metaspace的内存空间不再与堆连续，而是存在于本地内存中。

## 运行时常量池

运行时常量池对应class文件中的[`constant_pool`表](https://www.boris1993.com/projects/java/fundamentals/java-jvm-constant-pool.html)。

运行时常量池中包含了数值常量和属性的引用。每个运行时常量池的空间都会在类或接口被创建时生成，并且从方法区中分配空间。在创建运行时方法区时，如果申请的空间大于方法区可提供的空间，那么JVM会抛出`OutOfMemoryError`异常。

## 参考文档

\[^1\]: 《The Java Virtual Machine Specification (Java SE 8 Edition)》 - 2.5 Run-Time Data Areas  
\[^2\]: 面试官，Java8 JVM内存结构变了，永久代到元空间  
\[^3\]: [方法区 - JVM 运行时的数据区域](https://www.boris1993.com/projects/java/fundamentals/java-jvm-runtime-data-areas.html#jvm-method-area)  
\[^4\]: 方法区（永久区、元空间） - 深入理解JAVA虚拟机（内存模型+GC算法+JVM调优）  
\[^5\]: Java8内存模型—永久代(PermGen)和元空间(Metaspace)
