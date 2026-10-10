---
title: "在Java中为什么不推荐在代码中直接写字符串值"
source: https://www.boris1993.com/java-why-prefer-string-constants-over-literals.html
date: 2025-03-06
updated: 2026-10-10
tags: [Java]
categories: [学知识]
---

# 在Java中为什么不推荐在代码中直接写字符串值

前两天看到个问题，说在Java中，为什么不推荐在代码中直接写字符串值？我一开始能想到的就是代码的可维护性和可读性，但是总觉得这个问题没这么简单，肯定还有JVM虚拟机里面的优化。那既然现在大语言模型这么发达，我觉得可以让它给我讲讲。

## ChatGPT的回答

> 在 Java 代码中直接写字符串值（即 硬编码字符串）通常是不推荐的，主要从以下两个方面考虑：
> 
> 1.  代码可读性和可维护性
> 
> -   降低可读性：直接在代码中写死字符串值，容易让代码的意图变得不清晰，阅读时难以理解这个字符串的作用。
> -   难以维护：如果字符串值在多个地方使用，一旦需要修改，就必须逐个查找替换，容易遗漏或出错。
> -   不利于国际化（i18n）：如果程序需要支持多种语言，硬编码字符串必须手动提取，而如果使用常量或资源文件，则可以更方便地进行国际化处理。
> 
> 2.  JVM 内部优化
> 
> -   **字符串常量池优化**：JVM 会对字符串字面量进行\*\*字符串常量池（String Pool）\*\*处理，避免重复创建相同的字符串实例。但如果字符串值是动态拼接或new关键字创建的，就可能无法享受这种优化。
> -   **减少堆内存占用**：使用`String`变量或`final`常量，JVM 可能会进行`常量折叠`，减少运行时重复创建的开销，提高效率。
> -   **提高性能**：直接写字符串字面量可能会在某些情况下导致 JVM 额外分配对象，而使用`final`常量或`intern()`方法可以优化内存占用。

果然除了可读性考虑，JVM也会对字符串常量做一些优化。但，什么是`常量折叠`？

## 常量折叠

追问ChatGPT什么是常量折叠后，它是这么回到我的：

> 常量折叠（Constant Folding） 是 JVM 编译优化（Just-In-Time Compilation, JIT） 或 Java 编译器（javac）在编译阶段执行的一种优化策略，它会在 编译期 计算出可以确定的常量表达式，并用计算结果替换原始表达式，从而减少运行时的计算开销。
> 
> 1.  常量折叠的作用
> 
> 常量折叠的核心目标是提前计算出能确定的值，避免运行时重复计算，从而提高性能。例如，以下代码：
> 
> ```java
> public class Test {
>    public static void main(String[] args) {
>        int a = 2 * 3;  // 计算常量表达式
>        System.out.println(a);
>    }
> }
> ```
> 
> 在编译阶段，JVM 会将`2 * 3`计算为`6`，然后直接将`6`存入字节码，而不是在运行时执行乘法计算。
> 
> 2.  常量折叠的典型场景
> 
> （前略）
> 
> （2）字符串常量优化  
> 如果字符串是常量，编译器会在编译时进行字符串拼接优化，避免运行时的`StringBuilder`操作：
> 
> ```java
> public class StringConstantFolding {
>    public static void main(String[] args) {
>        final String s1 = "Hello, ";
>        final String s2 = "World!";
>        String s3 = s1 + s2;  // 直接折叠为 "Hello, World!"
>        System.out.println(s3);
>    }
> }
> ```
> 
> 优化后的字节码相当于：
> 
> ```java
> String s3 = "Hello, World!";
> ```
> 
> （后略）
> 
> 3.  什么时候不会发生常量折叠？  
>     （1）变量不是`final` - 如果变量不是`final`，编译器无法确定其值，因此不会进行常量折叠  
>     （2）值在运行时才能确定 - 如果值依赖于外部输入、方法调用、对象实例等，JVM 不能在编译时优化

也就是说，如果在代码里这样写：

```java
public class Test {
    private static final String HELLO = "hello, ";
    private static final String WORLD = "world";

    public static void main(String[] args) {
        System.out.println(HELLO + WORLD);
    }
}
```

那么实际上JVM就会给你优化成：

```java
public class Test{
    private static final String HELLO = "hello, ";
    private static final String WORLD = "world";

    public static void main(String[] args) {
        System.out.println("hello, world");
    }
}
```

将这段代码编译后再用`javap`反编译，可以看到这样的结果：

```plaintext
Classfile /Users/boris1993/Developer/java/leetcode/src/Test.class
  Last modified Mar 6, 2025; size 523 bytes
  SHA-256 checksum 3443c8e0a13cd862c0195b772bb4290bde7e1d07b50818d531c2b5d66d665270
  Compiled from "Test.java"
public class Test
  minor version: 0
  major version: 65
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #13                         // Test
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 2, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Fieldref           #8.#9          // java/lang/System.out:Ljava/io/PrintStream;
   #8 = Class              #10            // java/lang/System
   #9 = NameAndType        #11:#12        // out:Ljava/io/PrintStream;
  #10 = Utf8               java/lang/System
  #11 = Utf8               out
  #12 = Utf8               Ljava/io/PrintStream;
  #13 = Class              #14            // Test
  #14 = Utf8               Test
  #15 = String             #16            // hello, world
  #16 = Utf8               hello, world
  #17 = Methodref          #18.#19        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #18 = Class              #20            // java/io/PrintStream
  #19 = NameAndType        #21:#22        // println:(Ljava/lang/String;)V
  #20 = Utf8               java/io/PrintStream
  #21 = Utf8               println
  #22 = Utf8               (Ljava/lang/String;)V
  #23 = Utf8               HELLO
  #24 = Utf8               Ljava/lang/String;
  #25 = Utf8               ConstantValue
  #26 = String             #27            // hello,
  #27 = Utf8               hello,
  #28 = Utf8               WORLD
  #29 = String             #30            // world
  #30 = Utf8               world
  #31 = Utf8               Code
  #32 = Utf8               LineNumberTable
  #33 = Utf8               main
  #34 = Utf8               ([Ljava/lang/String;)V
  #35 = Utf8               SourceFile
  #36 = Utf8               Test.java
{
  public Test();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #15                 // String hello, world
         5: invokevirtual #17                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
}
SourceFile: "Test.java"
```

首先可以看到，在常量池中有这么两行代码：

```plaintext
#15 = String             #16            // hello, world
#16 = Utf8               hello, world
```

证明代码中`HELLO + WORLD`的值已经被编译器优化成了一个计算好的常量。在`main`方法的字节码中也可以看到

```plaintext
public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: (0x0009) ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=1, args_size=1
       0: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #15                 // String hello, world
       5: invokevirtual #17                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
```

JVM直接加载了”hello, world”这个字符串，而不是调用了`InvokeDynamic #0:makeConcatWithConstants:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;`在运行时进行字符串拼接。

## 总结

综上所述，使用字符串常量而不是直接写字符串值，除了让代码更可读更易于维护之外，我感觉最主要的优点就是针对字符串常量的拼接结果也会被放到常量池中，避免了在运行时反复创建新的字符串对象，造成性能浪费。
