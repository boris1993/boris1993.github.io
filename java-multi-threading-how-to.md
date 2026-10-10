---
title: "Java如何创建和运行多线程"
source: https://www.boris1993.com/java-multi-threading-how-to.html
date: 2019-02-28
updated: 2026-10-10
tags: [Java, 多线程]
categories: [学知识]
---

# Java如何创建和运行多线程

本文通过一个简单的示例，介绍一下在Java中如何创建和运行多线程，以及我在学习过程中遇到的问题。包括：

-   如何实现多线程
-   如何在线程间共享资源
-   共享资源时可能出现的问题

# 多线程的实现方法

多线程有三种实现方式：

1.  继承`Thread`类，并实现其`run()`方法；
2.  实现`Runnable`接口，并实现其`run()`方法；
3.  和实现`Callable`接口，并实现其`run()`方法。

通常来说，我们会通过实现`Runnable`接口来实现多线程，因为继承`Thread`类可能会有多继承的问题，而实现接口则没有这方面的影响。

下面示例会创建一个`MyThread`的类来实现，然后在`main()`中运行。

## 继承`Thread`类

```java
public class MyThread extends Thread {

    private int ticketCount = 5;
    private String threadName;

    public MyThread(String threadName) {
        this.threadName = threadName;
    }

    @Override
    public void run() {
        while (ticketCount > 0) {
            System.out.println(threadName + " has " + ticketCount-- + " tickets");
        }
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        new MyThread("thread1").start();
        new MyThread("thread2").start();
        new MyThread("thread3").start();
    }
}
```

## 实现`Runnable`接口

```java
public class MyThread implements Runnable {

    private int ticketCount = 5;
    private String threadName;

    public MyThread(String threadName) {
        this.threadName = threadName;
    }

    @Override
    public void run() {
        while (ticketCount > 0) {
            System.out.println(threadName + " has " + ticketCount-- + " tickets");
        }
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        new Thread(new MyThread("thread1")).start();
        new Thread(new MyThread("thread2")).start();
        new Thread(new MyThread("thread3")).start();

    }
}
```

## 实现`Callable`接口

\*\* TODO: 这东西看起来好像有点复杂，在这里先占个坑，改日单开一篇记录学习过程 \*\*

# 执行`start()`方法与执行`run()`方法的区别

实际上，唯一合法的运行多线程的方式，是调用`start()`方法，但是为什么不能调用`run()`方法呢？

因为`start()`方法会开辟一个新的线程，并且在新的线程中调用目标的`run()`方法。但是直接调用`run()`则不会创建新的线程，而是像调用其他任何一个方法那样，他将会在当前线程中执行。

这么说可能有些生涩，那么还是通过上面的例子来帮助理解。

在调用了`start()`方法后，程序的输出是这样子的，注意观察每行输出是由哪个线程写出来的：

```text
thread1 has 5 tickets
thread3 has 5 tickets
thread2 has 5 tickets
thread3 has 4 tickets
thread1 has 4 tickets
thread3 has 3 tickets
thread2 has 4 tickets
thread3 has 2 tickets
thread1 has 3 tickets
thread3 has 1 tickets
thread2 has 3 tickets
thread1 has 2 tickets
thread2 has 2 tickets
thread1 has 1 tickets
thread2 has 1 tickets
```

可见输出是乱序的。然而调用`run()`方法之后，输出变成了这样：

```text
thread1 has 5 tickets
thread1 has 4 tickets
thread1 has 3 tickets
thread1 has 2 tickets
thread1 has 1 tickets
thread2 has 5 tickets
thread2 has 4 tickets
thread2 has 3 tickets
thread2 has 2 tickets
thread2 has 1 tickets
thread3 has 5 tickets
thread3 has 4 tickets
thread3 has 3 tickets
thread3 has 2 tickets
thread3 has 1 tickets
```

看起来像是三个线程按照创建的顺序依次执行，但实际上只是先后调用了它们三个的`run()`方法而已，并没有新的线程被创建出来。

# 多线程共享资源

上文中卖票这个例子，都是开了三个线程，各卖各的票，但是实际上它们应该是从同一组票池中卖票。接下来，就把例子修改一下，让这三个线程共享资源。

```java
public class MyThread implements Runnable {

    private int ticketCount = 20;
    private String threadName;

    public MyThread(String threadName) {
        this.threadName = threadName;
    }

    @Override
    public void run() {
        while (ticketCount > 0) {
            // Thread.currentThread().getName() 打印出正在执行的线程的名字
            System.out.println(Thread.currentThread().getName() + " has " + ticketCount-- + " tickets");
        }
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        MyThread myThread = new MyThread("MyThread");

        Thread t1 = new Thread(myThread);
        Thread t2 = new Thread(myThread);
        Thread t3 = new Thread(myThread);

        t1.start();
        t2.start();
        t3.start();

    }
}
```

# 为什么用`Runnable`而不用`Thread`

`Thread(Runnable target)`的JavaDoc中，`target`参数的描述是这么写的：

> the object whose run method is invoked when this thread is started

以及`Thread#run()`是这样写的：

```java
public void run() {
    if (target != null) {
        target.run();
    }
}
```

同时`run()`的JavaDoc有如下描述：

> If this thread was constructed using a separate Runnable run object, then that Runnable object’s run method is called.

说明，在将一个`Runnable`对象赋给一个或多个`Thread`后，这些`Thread`调用的都是这一个`Runnable`对象的`run()`方法，所操作的数据也是这一个`Runnable`对象里面的数据。

依旧用例子说话。

在上一节的代码的`t1.start()`这一行打个断点，看看这三个线程的信息。

根据上面的JavaDoc，这里特别关注线程的`target`属性。

![Thread target to the same Runnable](https://blog-static.boris1993.com/java-multi-threading-how-to/thread-target-to-same-runnable.png)

可见，这三个`Thread`都使用了`MyThread@534`这个对象。也就是说，这三个线程都调用了`MyThread@534`的`run()`方法，并且在操作`MyThread@534`这个对象的成员变量。

然后，换成继承`Thread`的形式：

```java
public class MyThread extends Thread {
    private int ticketCount = 20;
    private String threadName;

    public MyThread(String threadName) {
        this.threadName = threadName;
    }

    @Override
    public void run() {
        while (ticketCount > 0) {
            System.out.println(Thread.currentThread().getName() + " has " + ticketCount-- + " tickets");
        }
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        MyThread t1 = new MyThread("MyThread1");
        MyThread t2 = new MyThread("MyThread2");
        MyThread t3 = new MyThread("MyThread3");

        t1.start();
        t2.start();
        t3.start();
    }
}
```

同样，在`t1.start()`上打断点，得到结果如下：

![Threads running separately](https://blog-static.boris1993.com/java-multi-threading-how-to/thread-running-separately.png)

可以发现，这三个`Thread`不止没有`target`，甚至它们的成员变量都是各自有一份，何谈线程之间共享。

# 多线程的同步问题

将[多线程共享资源](#%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%85%B1%E4%BA%AB%E8%B5%84%E6%BA%90)这一节的代码执行，得到了这样的输出：

```text
Thread-0 has 20 tickets
Thread-2 has 19 tickets
Thread-1 has 19 tickets
Thread-2 has 17 tickets
Thread-0 has 18 tickets
Thread-2 has 15 tickets
Thread-1 has 16 tickets
Thread-2 has 13 tickets
Thread-0 has 14 tickets
Thread-2 has 11 tickets
Thread-1 has 12 tickets
Thread-0 has 10 tickets
Thread-2 has 9 tickets
Thread-0 has 7 tickets
Thread-1 has 8 tickets
Thread-0 has 5 tickets
Thread-2 has 6 tickets
Thread-0 has 3 tickets
Thread-1 has 4 tickets
Thread-0 has 1 tickets
Thread-2 has 2 tickets
```

鞥？第二行和第三行好像不太对劲？线程1和线程2把同一张票重复卖了两次？果然出现了线程的同步问题了。

发生这个问题的原因是，Java中的自增、自减不是线程安全的。一个自增自减操作，实际上包含了三步：

1.  获取变量当前的值
2.  为该值加1或减1
3.  写回新值

那么要解决这个问题，就需要加锁，来保证“读-算-写”这个操作具有原子性，或者使用`AtomicInteger`类提供的原子操作。

## 使用`synchronized`关键字加锁

尝试使用`synchronized`关键字给`run()`方法加锁，代码修改如下：

```java
public class MyRunnable implements Runnable {

    private int ticketCount = 20;

    @Override
    public synchronized void run() {
        System.out.println(Thread.currentThread().getName() + " started.");

        while (true) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            if (ticketCount > 0) {
                System.out.println(Thread.currentThread().getName() + " has " + ticketCount-- + " tickets");
            } else {
                break;
            }
        }

        System.out.println(Thread.currentThread().getName() + " stopped.");
    }
}
```

运行后得到如下结果：

```text
Thread-0 started.
Thread-0 has 20 tickets
Thread-0 has 19 tickets
Thread-0 has 18 tickets
Thread-0 has 17 tickets
Thread-0 has 16 tickets
Thread-0 has 15 tickets
Thread-0 has 14 tickets
Thread-0 has 13 tickets
Thread-0 has 12 tickets
Thread-0 has 11 tickets
Thread-0 has 10 tickets
Thread-0 has 9 tickets
Thread-0 has 8 tickets
Thread-0 has 7 tickets
Thread-0 has 6 tickets
Thread-0 has 5 tickets
Thread-0 has 4 tickets
Thread-0 has 3 tickets
Thread-0 has 2 tickets
Thread-0 has 1 tickets
Thread-0 stopped.
Thread-2 started.
Thread-2 stopped.
Thread-1 started.
Thread-1 stopped.
```

可见`run()`方法被`Thread-0`上锁，被上锁的方法在释放锁前只能被一个线程所访问，`Thread-1`和`Thread-2`都在`Thread-0`执行结束并释放锁后才开始运行，并且也都进行了一次对`run()`的上锁-释放过程。

如果只对`ticketCount--`操作上锁呢？

```java
public class MyRunnable implements Runnable {

    private int ticketCount = 20;

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " started.");

        while (true) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 拿到了这里，而不是对run方法上锁
            synchronized (this) {
                if (ticketCount > 0) {
                    System.out.println(Thread.currentThread().getName() + " has " + ticketCount-- + " tickets");
                } else {
                    break;
                }
            }
        }

        System.out.println(Thread.currentThread().getName() + " stopped.");
    }
}
```

执行之后得到了这样的结果：

```text
Thread-0 started.
Thread-2 started.
Thread-1 started.
Thread-0 has 20 tickets
Thread-2 has 19 tickets
Thread-1 has 18 tickets
Thread-0 has 17 tickets
Thread-1 has 16 tickets
Thread-2 has 15 tickets
Thread-0 has 14 tickets
Thread-1 has 13 tickets
Thread-2 has 12 tickets
Thread-1 has 11 tickets
Thread-0 has 10 tickets
Thread-2 has 9 tickets
Thread-1 has 8 tickets
Thread-0 has 7 tickets
Thread-2 has 6 tickets
Thread-2 has 5 tickets
Thread-0 has 4 tickets
Thread-1 has 3 tickets
Thread-2 has 2 tickets
Thread-1 has 1 tickets
Thread-0 stopped.
Thread-2 stopped.
Thread-1 stopped.
```

三个线程在结束休眠后开始竞争锁，得到锁的线程操作了`ticketCount`，然后释放了锁。

## 原子操作

这次尝试将`ticketCount`换成`AtomicInteger`类型，并且使用`AtomicInteger#getAndDecrement()`方法进行原子的自减计算，修改后的代码如下：

```java
public class MyRunnable implements Runnable {

    private AtomicInteger ticketCount = new AtomicInteger(20);

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " started.");

        while (true) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            if (ticketCount.get() > 0) {
                System.out.println(Thread.currentThread().getName() + " has " + ticketCount.getAndDecrement() + " tickets");
            } else {
                break;
            }
        }

        System.out.println(Thread.currentThread().getName() + " stopped.");
    }
}
```

`main()`方法内容依旧不变，运行之后出现了这样的结果：

```text
Thread-1 started.
Thread-2 started.
Thread-0 started.
Thread-1 has 19 tickets
Thread-0 has 18 tickets
Thread-2 has 20 tickets
Thread-1 has 17 tickets
Thread-0 has 16 tickets
Thread-2 has 15 tickets
Thread-1 has 13 tickets
Thread-2 has 12 tickets
Thread-0 has 14 tickets
Thread-0 has 11 tickets
Thread-2 has 10 tickets
Thread-1 has 9 tickets
Thread-1 has 8 tickets
Thread-0 has 6 tickets
Thread-2 has 7 tickets
Thread-2 has 4 tickets
Thread-0 has 3 tickets
Thread-1 has 5 tickets
Thread-2 has 2 tickets
Thread-0 has 1 tickets
Thread-1 stopped.
Thread-2 stopped.
Thread-0 stopped.
```

虽然没有了脏读，但是线程的执行顺序也无法保证，如果要求线程定序执行，这样就不行了。
