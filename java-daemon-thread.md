---
title: "快速理解Java中的守护线程"
source: https://www.boris1993.com/java-daemon-thread.html
date: 2023-04-15
updated: 2026-10-10
tags: [Java, 多线程, 守护线程, 面试, daemon thread]
categories: [学知识]
---

# 快速理解Java中的守护线程

面试的时候被问到“了解不了解守护线程”，可本菜别提了解了，听都没听说过。所以面试之后赶紧上网冲浪学习起来。

## 什么是守护线程

首先我们要知道，Java中存在两种线程，用户线程和守护线程。

用户线程就是我们平时最常用到的那种线程，它属于高优先级线程，JVM在退出前会等待所有用户线程完成。

守护线程则是低优先级线程，它的作用是为用户线程提供服务，不会阻止JVM退出（不能100%保证，不良的代码仍会阻止JVM退出）。

> 即Thread#setDaemon(boolean)方法的JavaDoc提到的  
> The Java Virtual Machine exits when the only threads running are all daemon threads.

## 守护线程能干什么

常见的场景有垃圾回收、释放未使用的对象的内存、从缓存中释放不需要的条目等等。

## 怎么创建守护线程

调用`Thread#setDaemon(boolean)`方法就可以让一个线程变成守护线程，如：

```java
import java.util.concurrent.TimeUnit;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Main thread exited.");
        }));

        Thread thread = new Thread(() -> {
            try {
                while (true) {
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println("Thread is running...");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println("Thread exited.");
            }
        });

        // 一定要在Thread#start()前调用，否则会抛IllegalThreadStateException异常
        thread.setDaemon(true);

        thread.start();

        TimeUnit.SECONDS.sleep(5);

        System.out.println("Main thread exiting.....");
    }
}
```

将上面代码运行后，可以在控制台看到如下输出：

```text
Thread is running...
Thread is running...
Thread is running...
Thread is running...
Main thread exiting.....
Main thread exited.
```

可见，虽然线程中执行的是一个死循环，但是JVM在结束时，不会理会`thread`还在运行，直接就退出了。同样还可以看到的是，`finally`块并没有执行，所以在守护线程中不要使用任何需要关闭的资源，比如打开文件，因为守护线程退出的时候，它没有任何机会来关闭文件，就会导致数据丢失（我感觉就像`kill -9`掉一个正在内存中处理文件内容的进程，`SIGKILL`信号不会给这个进程任何善后的时间）。
