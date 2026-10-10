---
title: "Java线程池"
source: https://www.boris1993.com/java-multi-threading-thread-pool.html
date: 2019-03-04
updated: 2026-10-10
tags: [Java, 多线程]
categories: [学知识]
---

# Java线程池

就像数据库连接可以使用连接池管理一样，Java中的线程也可以使用线程池来管理。本文介绍在Java中如何使用线程池，以及有哪些线程池。

# 为什么需要线程池

每个线程的创建和销毁，都会消耗一定的系统资源，尤其在高并发的系统中，频繁创建和销毁线程会造成大量的资源浪费。

那么，为了避免频繁的创建和销毁线程，就可以在系统启动时，预先创建好一定数量的线程，并将其交由线程调度器管理，这就是线程池。

# 怎么使用线程池

依旧是用一个示例来演示。

```java
public class MyRunnable implements Runnable {

    private int ticketCount = 20;

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " started.");

        while (ticketCount > 0) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            synchronized (gthis) {
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

```java
public class Main {

    public static void main(String[] args) {
        // 创建一个线程池
        ExecutorService executorService = Executors.newCachedThreadPool();

        System.out.println("Thread pool created");

        MyRunnable myRunnable = new MyRunnable();

        System.out.println("Assigning jobs to thread pool");

        // 向线程池提交任务
        executorService.exeute(myRunnable);
        executorService.execute(myRunnable);

        // 在所有线程都完成工作后，线程池会继续等待新的工作任务
        // 所以如果需要程序在完成后退出，需要显式关闭线程池
        executorService.shutdown();

        while (!executorService.isTerminated()) { }

        System.out.println("Thread pool is down");
    }
}
```

运行后得到如下结果：

```text
Thread pool created
Assigning jobs to thread pool
pool-1-thread-2 started.
pool-1-thread-1 started.
pool-1-thread-1 has 20 tickets
pool-1-thread-2 has 19 tickets
pool-1-thread-1 has 18 tickets
pool-1-thread-2 has 17 tickets
pool-1-thread-1 has 16 tickets
pool-1-thread-2 has 15 tickets
pool-1-thread-1 has 14 tickets
pool-1-thread-2 has 13 tickets
pool-1-thread-2 has 12 tickets
pool-1-thread-1 has 11 tickets
pool-1-thread-1 has 10 tickets
pool-1-thread-2 has 9 tickets
pool-1-thread-1 has 8 tickets
pool-1-thread-2 has 7 tickets
pool-1-thread-2 has 6 tickets
pool-1-thread-1 has 5 tickets
pool-1-thread-2 has 4 tickets
pool-1-thread-1 has 3 tickets
pool-1-thread-2 has 2 tickets
pool-1-thread-1 has 1 tickets
pool-1-thread-1 stopped.
pool-1-thread-2 stopped.
Thread pool is down
```

# 几种线程池的简介

## Single thread pool

是用`Executors.newSingleThreadExecutor()`创建。

该线程池仅有一个线程，并拥有一个无边界的队列。提交到队列中的任务将会按顺序执行。如果当前线程在执行过程中出现错误而被终止，那么线程池会创建一个新的线程并继续执行队列中后续的任务。

与使用`Executors.newFixedThreadPool(1)`创建的线程池不同的是，我们不能为single thread pool分配更多的线程数。

## Fixed thread pool

使用`Executors.newFixedThreadPool(int nThreads)`创建。

该线程池维护着固定数量的线程(nThreads个)，在任何时间只允许最多nThreads个线程执行任务，多出来的任务将会在队列中等待，直到有空闲的线程出现。如果其中一个线程在执行过程中因为错误而异常退出，则线程池会立刻创建一个新的线程并执行后续的任务。

该线程池在显式关闭(`ExecutorService#shutdown`)前将一直存在。

## Work stealing pool

使用`Executors.newWorkStealingPool(int parallelism)`或`Executors.newWorkStealingPool()`创建。

该线程池无法保证各个被提交的任务将会以何种顺序执行。

### `newWorkStealingPool(int parallelism)`

该方法将根据给定的“并行量(parallelism)”，来创建一个包含足够数量线程的线程池，并会使用多个队列来减少线程与队列的争抢。

“并行量”的值对应于最多允许参与执行任务的线程数量。但实际存在的线程数可能会动态的增减。

### `Executors.newWorkStealingPool()`

将所有的“可用的处理器”的数目作为“并行量”来创建线程池。

可用的处理器数量使用`Runtime.getRuntime().availableProcessors()`获取，其值等同于CPU中逻辑处理器的数量。

## Cached thread pool

使用`Executors.newCachedThreadPool()`创建。

当接收到新的任务后，线程池会根据有无可用线程，来决定使用线程池中的空闲线程，或者在线程池中创建新的线程。

如果线程池中有线程空置超过60秒，则该线程就会被终止并从线程池中移除。

## Scheduled thread pool

使用`Executors.newScheduledThreadPool(int corePoolSize)`创建。`corePoolSize`为线程池中保持的线程数。

该线程池可以指定一个延迟，或指定一个周期，并按照这个计划执行任务。

# 线程池是如何工作的

那么，在调用`ExecutorService#execute(Runnable)`之后，线程池究竟做了些什么呢？

在`ThreadPoolExecutor#execute(Runnable)`方法中，有这样一段注释：

> Proceed in 3 steps:
> 
> 1.  If fewer than corePoolSize threads are running, try to  
>     start a new thread with the given command as its first  
>     task. The call to addWorker atomically checks runState and  
>     workerCount, and so prevents false alarms that would add  
>     threads when it shouldn’t, by returning false.
> 2.  If a task can be successfully queued, then we still need  
>     to double-check whether we should have added a thread  
>     (because existing ones died since last checking) or that  
>     the pool shut down since entry into this method. So we  
>     recheck state and if necessary roll back the enqueuing if  
>     stopped, or start a new thread if there are none.
> 3.  If we cannot queue task, then we try to add a new  
>     thread. If it fails, we know we are shut down or saturated  
>     and so reject the task.

也就是说：

```

flowchart TD
check_running_threads_number{检查正在运行的线程数是否小于corePoolSize};
start_new_thread("调用addWorker尝试开启一个新的线程，并将这个runnable作为第一个task交给这个线程");
try_enqueue_the_task{"尝试将任务加入队列，并重新调用addWorker再次检查能否加入新的worker"};
reject_the_task("此时线程池已经饱和，或者正在被关闭，所以拒绝掉这个task");

开始 --> check_running_threads_number
check_running_threads_number -- 是 --> start_new_thread
start_new_thread -- 成功 --> 结束
start_new_thread -- 失败 --> try_enqueue_the_task
check_running_threads_number -- 否 --> try_enqueue_the_task
try_enqueue_the_task -- 成功 --> 结束
try_enqueue_the_task -- 失败 --> reject_the_task
reject_the_task --> 结束
```

这里要注意一个比较容易引起误解的点，就是在`core pool size < 当前正在运行的线程数量 < max pool size`时，如果有新的任务进来，那么这个任务首先会被放入队列。仅当队列满了的时候，线程池才会为这个新来的任务创建新的线程。

# 参考文章

Thread pools and work queues  
Core pool size vs maximum pool size in ThreadPoolExecutor - StackOverflow
