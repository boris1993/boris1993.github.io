---
title: "Java的内部类和private修饰符"
source: https://www.boris1993.com/java-inner-class-and-private-identifier.html
date: 2020-02-24
updated: 2026-10-10
tags: [Java, 内部类]
categories: [学知识]
---

# Java的内部类和private修饰符

一个`private`属性只能被它所在的类访问，这件事地球人都知道。但是，你有没有想过，这条规则有没有可能在某种情况下，会变得不成立？

本文将通过一个小例子，来演示怎么让`private`修饰符“失效”，以及它为什么会“失效”。

## 示例代码

废话不多说，先写一段代码：

```java
public class OuterClass {
    private String outerClassName = "outerClass";

    public class InnerClass {
        public void printOuterClassName() {
            System.out.println(outerClassName);
        }
    }

    public static void main(String[] args) {
        OuterClass outerClass = new OuterClass();
        OuterClass.InnerClass innerClass = outerClass.new InnerClass();

        innerClass.printOuterClassName();
    }
}
```

上面的代码是不是感觉有一丝异样？为什么在内部类里，能直接访问到外部类的`private`属性？难道`private`修饰符真的“失效”了？

别急，待我们把这个class反编译了，从字节码层面来看看它到底有什么猫腻。毕竟，字节码可不会骗人。

## 反编译外部类

```bash
$ javap -c OuterClass.class
Compiled from "OuterClass.java"
public class com.boris1993.OuterClass {
  public com.boris1993.OuterClass();
    Code:
       0: aload_0
       1: invokespecial #2                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: ldc           #3                  // String outerClass
       7: putfield      #1                  // Field outerClassName:Ljava/lang/String;
      10: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #4                  // class com/boris1993/OuterClass
       3: dup
       4: invokespecial #5                  // Method "<init>":()V
       7: astore_1
       8: new           #6                  // class com/boris1993/OuterClass$InnerClass
      11: dup
      12: aload_1
      13: dup
      14: invokevirtual #7                  // Method java/lang/Object.getClass:()Ljava/lang/Class;
      17: pop
      18: invokespecial #8                  // Method com/boris1993/OuterClass$InnerClass."<init>":(Lcom/boris1993/OuterClass;)V
      21: astore_2
      22: aload_2
      23: invokevirtual #9                  // Method com/boris1993/OuterClass$InnerClass.printOuterClassName:()V
      26: return

  static java.lang.String access$000(com.boris1993.OuterClass);
    Code:
       0: aload_0
       1: getfield      #1                  // Field outerClassName:Ljava/lang/String;
       4: areturn
}
```

有没有发现，78行出现了一个我们没有写过的方法`access$000`？而且从注释来看，它接受一个`OuterClass`类型的参数，而且返回的正是外部类的`outerClassName`的值。

既然我们没定义这个方法，那就是编译器偷偷的给咱整了点活。至于为啥编译器要这么干，结合上面这个例子，也不难猜出来：这就是给内部类访问它的`private`属性用的。

## 反编译内部类

但是咱不能光猜啊，咱还得有证据。证据哪来？当然是内部类的字节码。

```bash
$ javap -c OuterClass$InnerClass.class
Compiled from "OuterClass.java"
public class com.boris1993.OuterClass$InnerClass {
  final com.boris1993.OuterClass this$0;

  public com.boris1993.OuterClass$InnerClass(com.boris1993.OuterClass);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #1                  // Field this$0:Lcom/boris1993/OuterClass;
       5: aload_0
       6: invokespecial #2                  // Method java/lang/Object."<init>":()V
       9: return

  public void printOuterClassName();
    Code:
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: aload_0
       4: getfield      #1                  // Field this$0:Lcom/boris1993/OuterClass;
       7: invokestatic  #4                  // Method com/boris1993/OuterClass.access$000:(Lcom/boris1993/OuterClass;)Ljava/lang/String;
      10: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: return
}
```

嗯，果然没错，在第20行这一条指令里，它调用了上面我们看到的那个`access$000()`方法。
