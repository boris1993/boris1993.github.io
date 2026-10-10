---
title: "Java源码阅读 - Timer"
source: https://www.boris1993.com/java-read-src-timer.html
date: 2023-12-19
updated: 2026-10-10
tags: [Java, Timer]
categories: [学知识]
---

# Java源码阅读 - Timer

前段时间拿`ConcurrentHashMap`和`Timer`简单整了个带过期功能的缓存，那用都用了，顺便就看看`Timer`这玩意是咋实现的。

## 代码示例

这段代码的背景是，我们想要把已经验证通过的token缓存起来，下次遇到同样的token就不需要再浪费CPU做重复的校验，直接从缓存中返回解析好的token对象就行。而因为token会过期，所以要每秒检查一次缓存里面是否有token过期，并批量从缓存中删除。

验证和缓存token那部份不是本文重点，就不写了。清除过期缓存的`Timer`就是这样实现的：

```java
// 定义一个Timer
private final Timer purgeCacheTimer = new Timer();

// 在构造方法里面注册一个定时任务
purgeCacheTimer.scheduleAtFixedRate(new TimerTask() {
    @Override
    public void run() {
        final List<String> expiredTokenList = new ArrayList<>();

        for (final Map.Entry<String, Object> token : validatedTokenCache.entrySet()) {
            if (isTokenExpired(token.getValue())) {
                expiredTokenList.add(token.getKey());
            }
        }

        expiredTokenList.forEach(validatedTokenCache::remove);
    }
}, 1000, 1000);
```

## `Timer`的核心数据结构

点进`Timer`类就能在开头看到两个属性：`queue`和`thread`。

```java
/**
 * The timer task queue.  This data structure is shared with the timer
 * thread.  The timer produces tasks, via its various schedule calls,
 * and the timer thread consumes, executing timer tasks as appropriate,
 * and removing them from the queue when they're obsolete.
 */
private final TaskQueue queue = new TaskQueue();

/**
 * The timer thread.
 */
private final TimerThread thread = new TimerThread(queue);
```

`queue`所属的`TaskQueue`就是这个`Timer`用来存储任务的队列，里面是把一个数组处理成了一个优先队列。`thread`所属的`TimerThread`类则是`Timer`的核心，负责任务调度。

## 任务是怎么添加进队列的

先看看给`Timer`添加任务会发生什么。点进`scheduleAtFixedRate`可以看到这么一段代码：

```java
public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, System.currentTimeMillis()+delay, period);
}
```

跟着点进`sched`可以看到这么一段：

```java
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");

    // Constrain value of period sufficiently to prevent numeric
    // overflow while still being effectively infinitely large.
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");

        synchronized(task.lock) {
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            task.nextExecutionTime = time;
            task.period = period;
            task.state = TimerTask.SCHEDULED;
        }

        queue.add(task);
        if (queue.getMin() == task)
            queue.notify();
    }
}
```

看下来其实就做了两件事：向`queue`里面增加一个`TimerTask`对象，然后检查队列里面即将执行的task，如果是刚添加的这个task那就唤醒任务。

## 任务是怎么被调度的

点进`TimerThread`首先可以看到它的`run`方法：

```java
public void run() {
    try {
        mainLoop();
    } finally {
        // Someone killed this Thread, behave as if Timer cancelled
        synchronized(queue) {
            newTasksMayBeScheduled = false;
            queue.clear();  // Eliminate obsolete references
        }
    }
}
```

因为`TimerThread`继承自`Thread`类，所以`run`方法就是它的入口点。在`Timer`的构造函数中就会执行`thread.start()`操作，所以这个调度线程在`Timer`被创建后就会开始运行。

继续看看`mainLoop`里面干了些啥。

```java
/**
 * The main timer loop.  (See class comment.)
 */
private void mainLoop() {
    while (true) {
        try {
            TimerTask task;
            boolean taskFired;
            synchronized(queue) {
                // Wait for queue to become non-empty
                while (queue.isEmpty() && newTasksMayBeScheduled)
                    queue.wait();
                if (queue.isEmpty())
                    break; // Queue is empty and will forever remain; die

                // Queue nonempty; look at first evt and do the right thing
                long currentTime, executionTime;
                task = queue.getMin();
                synchronized(task.lock) {
                    if (task.state == TimerTask.CANCELLED) {
                        queue.removeMin();
                        continue;  // No action required, poll queue again
                    }
                    currentTime = System.currentTimeMillis();
                    executionTime = task.nextExecutionTime;
                    if (taskFired = (executionTime<=currentTime)) {
                        if (task.period == 0) { // Non-repeating, remove
                            queue.removeMin();
                            task.state = TimerTask.EXECUTED;
                        } else { // Repeating task, reschedule
                            queue.rescheduleMin(
                                task.period<0 ? currentTime   - task.period
                                            : executionTime + task.period);
                        }
                    }
                }
                if (!taskFired) // Task hasn't yet fired; wait
                    queue.wait(executionTime - currentTime);
            }
            if (taskFired)  // Task fired; run it, holding no locks
                task.run();
        } catch(InterruptedException e) {
        }
    }
}
```

首先它检查队列是否为空，如果是空的，那么这个调度线程就会进入等待状态（这就是在添加任务之后要调用`notify()`方法的原因）；如果不仅队列是空的，而且队列不再接收新的任务，那么就说明这个`Timer`正在被销毁，所以直接退出循环。

如果队列非空，那么调度线程就从队列中取出最接近下一个执行时机的任务，然后安排下一次任务执行的时间。同时，在取出最近需要执行的任务后，调度线程会比较当前时间与计划执行时间，如果当前时间还没到，那么调度线程会`wait`到计划执行时间。最后，调用`TimerTask`的`run`方法，开始执行我们指定它做的事。

## 总结

说白了就是俩东西：

-   一个保存任务的队列，里面的任务按照执行时间排队
-   一个死循环，反复从队列里面取出最接近下一个执行时机的任务，执行它，并安排下一次的时机

这玩意优点就是简单粗暴而且轻量级，但因为它是单线程的，如果某个任务执行时间过长，那么后续任务有可能会受影响；而且因为系统调度和任务执行时间的不确定性，它不能提供高精度的定时任务服务；此外，如果其中某个任务抛出了未处理的异常，那么整个`Timer`都会挂掉，也会影响到同一个队列里面的其他任务。
