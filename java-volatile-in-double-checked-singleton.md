---
title: "volatile修饰符在双检锁单例模式中的作用"
source: https://www.boris1993.com/java-volatile-in-double-checked-singleton.html
date: 2023-12-10
updated: 2026-10-10
tags: [Java, volatile, singleton, 单例模式, 双检锁]
categories: [学知识]
---

# volatile修饰符在双检锁单例模式中的作用

在实现一个双检锁单例的时候，IDEA提示我要给`INSTANCE`实例加上`volatile`修饰符。当时并不明白为啥，所以选择相信IDE。但是还是那句话，不能知其然不知其所以然啊，自己写的代码，不能自己心里没底不是。于是乎我一顿网上冲浪，终于整明白了为啥双检单例必须要用`volatile`修饰符。

## 代码示例

这个单例类没什么好说的，就是一个平平无奇的双检锁单例实现。

```java
public class Singleton {
    private static Singleton INSTANCE;

    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }

        return INSTANCE;
    }

    public void doSomething() {
        // Do something here
    }
}
```

而IDEA在外层的`if`上标了一个警告，并且建议我给`INSTANCE`变量加上`volatile`修饰符。

![](https://blog-static.boris1993.com/java-volatile-in-double-checked-singleton/suggestion-in-idea.png)

## 如果不加`volatile`会有什么问题

上面的代码，乍一看非常严谨，在发现`INSTANCE`是`null`的时候，就对其加锁并再检查一次，还是`null`的话就为它创建一个新的实例，最后返回它。但是看了一些文章之后发现，在多线程场景下，有可能出现虽然成功获取到`INSTANCE`，但在调用其中的方法时仍然抛出空指针异常的诡异情况。

比如有这样一个场景，`Thread 1`和`Thread 2`同时请求了`Singleton#getInstance()`方法，`Thread 1`执行到了第8行，开始实例化这个对象；而`Thread 2`执行到了第5行，开始检查`INSTANCE`是否为`null`。这个时候，有一定几率，虽然`Thread 2`检查到`INSTANCE`并不是`null`，但是调用`Singleton#doSomething()`方法的时候却会抛出空指针异常。

![](https://blog-static.boris1993.com/java-volatile-in-double-checked-singleton/instance-accessed-by-multi-threads.png)

造成这个问题的原因就是Java的指令重排。

在搞清楚`Thread 2`看到`INSTANCE`虽然不是`null`，却在方法调用的时候会抛空指针异常的原因之前，先要搞清楚实例化对象的时候，JVM到底干了什么。

JVM实例化一个对象的过程，大致可以分为这几步：

1.  JVM为这个对象分配一片内存
2.  在这片内存上初始化这个对象
3.  将这片内存的地址赋值给`INSTANCE`变量

因为把内存地址赋值给`INSTANCE`是最后一步，所以`Thread 1`在这一步执行之前，`Thread 2`对`INSTANCE == null`的判断一定为`true`，进而因为拿不到`Singleton`类的锁而被阻塞，直到`Thread 1`完成对`INSTANCE`变量的实例化。

但是，上面这三步它不是个原子操作，并且JVM可能会进行重排序，也就是说上面这三步可能被重排成

1.  JVM为这个对象分配一片内存
2.  将这片内存的地址赋值给`INSTANCE`变量
3.  在这片内存上初始化这个对象

你看，这问题就来了，如果在`Thread 1`做完第二步但没做第三步的时候，`Thread 2`开始检查`INSTANCE`是不是`null`就会得到`false`，然后就走到`return`，得到一个不完整的`INSTANCE`对象。这时候，虽然`INSTANCE`不是`null`，但同时它也没有完成初始化，所以`Thread 2`在调用`Singleton#doSomething()`方法的时候，就会抛出空指针异常。

这个问题的解决方案就是`volatile`修饰符，因为它可以禁止指令重排，所以在给`INSTANCE`加上`volatile`之后，JVM就会老老实实的先初始化好这个对象，再为`INSTANCE`赋值，这样多线程场景下每个线程得到的`INSTANCE`实例都会是一个初始化好了的`Singleton`对象。
