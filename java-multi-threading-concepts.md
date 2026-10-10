---
title: "Java多线程概念"
source: https://www.boris1993.com/java-multi-threading-concepts.html
date: 2019-03-03
updated: 2026-10-10
tags: [Java, 多线程]
categories: [学知识]
---

# Java多线程概念

本文记录一些Java多线程相关的概念性的知识。

# 线程的状态

![Java thread states](https://blog-static.boris1993.com/java-multi-threading-concepts/java-thread-states.png)

## 新建(New)

线程已被创建，但是尚未启动

## 可运行(Runnable)

此线程在JVM中正在运行

## 阻塞(Blocked)

此线程正在等待获取一个监视锁(monitor lock)，需要其他线程显式唤醒

## 等待(Waiting)

此线程正在无限期等待另一个线程完成某些工作

| 进入方法 | 退出方法 |
| --- | --- |
| `Object#wait()` | `Object#notify()`或`Object#notifyAll()` |
| `Thread#join()` | 被调用的线程执行完毕 |

## 限期等待(Timed waiting)

此线程正在有限期等待另一个线程完成某些工作

| 进入方法 | 退出方法 |
| --- | --- |
| `Thread.sleep()` | 设定的休眠时间结束 |
| `Object#wait(long timeout)` | 时间结束 / `Object#notify()` / `Object#notifyAll()` |
| `Thread#join(long millis)` | 时间结束 / 被调用的线程执行完毕 |

## 终止(Terminated)

线程结束

# 使用线程

见[Java 如何创建和运行多线程](https://www.boris1993.com/projects/java/fundamentals/java-multi-threading-how-to.html)

# 互斥同步

## `synchronized`

### 同步一个代码块

只作用于同一个对象，如多个`Thread`使用同一个`Runnable`时。一个线程若要使用此方法，则必须获得obj对象的锁。

```java
public void something() {
    synchronized (obj) {
        // do something
    }
}
```

### 同步一个方法

```java
// 只作用于同一个对象。
// 一个线程若要使用此方法，则必须获得该方法所在对象的锁
public void synchronized something() {
    // do something
}
```

```java
// 作用于整个类
// 一个线程若要使用此方法，则必须获得该方法所在类的锁
public void static synchronized aStaticMethod() {
    // do something
}
```

### 同步一个类

作用于整个类，即使两个线程使用同一个类的不同对象，也会进行同步。一个线程若要使用此方法，则必须获得该类的锁。

```java
public void something() {
    synchronized (SynchronizationExample.class) {
        // do something
    }
}
```

## `ReentrantLock`

`ReentrantLock`是`java.util.concurrent`包中的锁

```java
public class LockDemo implements Runnable {
    private Lock lock = new ReentrantLock();

    @Override
    public void run() {
        try {
            lock.lock();
            for (int i = 0; i < 10; i++) {
                System.out.println(i + " ");
            }
        } finally {
            lock.unlock();
        }
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        LockDemo lockDemo = new LockDemo();

        ExecutorService executorService = Executors.newCachedThreadPool();

        executorService.execute(lockDemo);
        executorService.execute(lockDemo);

    }
}
```

# 线程协作

## `Thread#join()`

在A线程中调用B线程的`join()`方法，会将当前线程挂起，直到目标线程结束。

```java
public class MyRunnable implements Runnable {

    private AtomicInteger ticketCount = new AtomicInteger(5);

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

```java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new MyRunnable());
        Thread t2 = new Thread(new MyRunnable());
        Thread t3 = new Thread(new MyRunnable());

        t1.start();

        // 主线程等待 t1 执行完毕
        t1.join();

        t2.start();
        t3.start();
    }
}
```

执行结果如下：

```text
Thread-0 started.
Thread-0 has 5 tickets
Thread-0 has 4 tickets
Thread-0 has 3 tickets
Thread-0 has 2 tickets
Thread-0 has 1 tickets
Thread-0 stopped.
Thread-1 started.
Thread-2 started.
Thread-1 has 5 tickets
Thread-2 has 5 tickets
Thread-2 has 4 tickets
Thread-1 has 4 tickets
Thread-2 has 3 tickets
Thread-1 has 3 tickets
Thread-2 has 2 tickets
Thread-1 has 2 tickets
Thread-1 has 1 tickets
Thread-2 has 1 tickets
Thread-1 stopped.
Thread-2 stopped.
```

## `wait()`，`notify()`，和`notifyAll()`

`wait()`将当前线程变为等待状态，`notify()`和`notifyAll()`将等待状态的线程唤醒。`wait()`方法必须在有锁(即`synchronized`)的代码块中执行。

当有多个线程处于等待状态时，`notify()`会任意选择一个线程来唤醒，选择的方式由JVM的实现来决定；而`notifyAll()`则会唤醒所有等待中的线程。

因为线程唤醒后，程序将会从`wait()`的下一条语句中开始执行，所以`wait()`方法应当总在`while`循环中调用，通过循环条件控制线程是否继续等待。

```java
public class WaitNotifyDemo {
    public synchronized void before() {
        System.out.println("before");
        notifyAll();
    }

    public synchronized void after() {
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("after");
    }
}
```

```java
public class Main {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newCachedThreadPool();
        WaitNotifyDemo waitNotifyDemo = new WaitNotifyDemo();

        // 调用after()后，遇到wait()进入等待状态
        executorService.execute(waitNotifyDemo::after);

        // 调用before()后，遇到`notifyAll()`，唤醒所有线程
        executorService.execute(waitNotifyDemo::before);
    }
}
```

执行结果：

```text
before
after
```
