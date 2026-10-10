---
title: "理解Java中的抽象队列同步器（AQS）"
source: https://www.boris1993.com/java-understand-aqs.html
date: 2024-01-17
updated: 2026-10-10
tags: [Java, Lock, AQS, 锁]
categories: [学知识]
---

# 理解Java中的抽象队列同步器（AQS）

最近项目里用到了些Lock，爬了些文了解到它们是基于`AbstractQueuedSynchronizer`（即`AQS`）实现的。那么，不如趁热打铁，看看里面是怎么工作的。

## 什么是AQS

`AbstractQueuedSynchronizer`，抽象队列同步器，是很多同步器（如`ReentrantLock`、`CountDownLatch`、`Semaphore`）等都是基于它实现的。

在AQS内部，它维护了一个FIFO队列，和一个`volatile`类型的变量`state`。FIFO队列用来实现多线程的排队工作，线程加锁失败时，这个线程就会被封装成一个`Node`节点放到队尾，然后当锁被释放后，队列头部的线程就会被唤醒并让它重新尝试获取锁；`state`变量用来记录锁的状态，如`Semaphore`的`permit`就是存在`state`里面的。

## 状态管理

上面说到，AQS使用一个`volatile`的`int`变量`state`来管理锁的状态，`state`为0时说明锁被释放，反之锁被持有。

AQS提供了三个方法来同步锁的状态：`getState()`，`setState(int newState)`和`compareAndSetState(int expect, int update)`。

```java
/**
 * The synchronization state.
 */
private volatile int state;

/**
 * Returns the current value of synchronization state.
 * This operation has memory semantics of a {@code volatile} read.
 * @return current state value
 */
protected final int getState() {
    return state;
}

/**
 * Sets the value of synchronization state.
 * This operation has memory semantics of a {@code volatile} write.
 * @param newState the new state value
 */
protected final void setState(int newState) {
    state = newState;
}

/**
 * Atomically sets synchronization state to the given updated
 * value if the current state value equals the expected value.
 * This operation has memory semantics of a {@code volatile} read
 * and write.
 *
 * @param expect the expected value
 * @param update the new value
 * @return {@code true} if successful. False return indicates that the actual
 *         value was not equal to the expected value.
 */
protected final boolean compareAndSetState(int expect, int update) {
    return U.compareAndSetInt(this, STATE, expect, update);
}
```

查看`setState`方法的引用，不难发现像`CountDownLatch`和`Semaphore`这些熟悉的身影。

![](https://blog-static.boris1993.com/java-understand-aqs/usage-of-set-state.png)

## FIFO队列 - 线程排队等待锁的地方

在AQS内部，未能成功获取锁的线程都会被包装成一个`Node`节点，然后放到FIFO队列尾部让它等待。

```java
// Node status bits, also used as argument and return values
static final int WAITING   = 1;          // must be 1
static final int CANCELLED = 0x80000000; // must be negative
static final int COND      = 2;          // in a condition wait

abstract static class Node {
    volatile Node prev;       // initially attached via casTail
    volatile Node next;       // visibly nonnull when signallable
    Thread waiter;            // visibly nonnull when enqueued
    volatile int status;      // written by owner, atomic bit ops by others
    // 略
}

/**
 * Head of the wait queue, lazily initialized.
 */
private transient volatile Node head;

/**
 * Tail of the wait queue. After initialization, modified only via casTail.
 */
private transient volatile Node tail;

/**
 * Enqueues the node unless null. (Currently used only for
 * ConditionNodes; other cases are interleaved with acquires.)
 */
final void enqueue(Node node) {
    if (node != null) {
        for (;;) {
            Node t = tail;
            node.setPrevRelaxed(t);        // avoid unnecessary fence
            if (t == null)                 // initialize
                tryInitializeHead();
            else if (casTail(t, node)) {
                t.next = node;
                if (t.status < 0)          // wake up to clean link
                    LockSupport.unpark(node.waiter);
                break;
            }
        }
    }
}
```

## Semaphore

`Semaphore`就是AQS的一个实现，从它的源码就能很容易看出来，它内部就是通过AQS的`state`来管理`permits`。

```java
public class Semaphore implements java.io.Serializable {
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;

    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    /**
     * NonFair version
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and nonfair fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and the given fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     * @param fair {@code true} if this semaphore will guarantee
     *        first-in first-out granting of permits under contention,
     *        else {@code false}
     */
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
}
```

## 与synchronized的区别

-   `synchronized`是一个Java内置的关键字，`AQS`扩展的各种锁则是通过Java代码实现的
-   `synchronzed`锁是自动获取和释放的，而`AQS`的锁需要手动获取和释放
-   像`ReentrantLock`还可以设置超时等特性，但`synchronized`不行
