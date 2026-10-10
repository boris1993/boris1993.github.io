---
title: "自己实现一个BlockingQueue并优化"
source: https://www.boris1993.com/enhancing-my-blocking-queue-implementation.html
date: 2025-03-26
updated: 2026-10-10
tags: [Java, BlockingQueue]
categories: [学知识]
---

# 自己实现一个BlockingQueue并优化

前阵看见个面试题

> 请实现一个泛型类BlockingQueue：
> 
> 1.  构造函数里指定队列容量
> 2.  void put(T item) 队列满了会阻塞，直到队列有空间
> 3.  T take()，队列空时会阻塞，直到队列有元素

我寻思做做看，结果写出来的答案我自己都看不下去，那干脆看看Java里面是咋实现的。

## 我的实现

先看看我写的玩意是啥德行吧。实话说这玩意看着都不像能正常工作的样子（虽然实际上它还真能用）

```java
public class BlockingQueue<T> {
    private final Object[] items;
    private final int maxSize;
    private volatile int count;

    public BlockingQueue(final int size) {
        this.items = new Object[size];
        this.maxSize = size;
    }

    public synchronized void put(final T item) throws InterruptedException {
        while (this.count >= maxSize) {
            wait();
        }

        while (true) {
            if (this.count < maxSize) {
                notifyAll();
                break;
            }
        }

        synchronized (this) {
            items[count] = item;
            count++;
        }
    }

    public synchronized T take() throws InterruptedException {
        while (count == 0) {
            wait();
        }

        while (true) {
            if (count > 0) {
                notifyAll();
                break;

            }
        }

        final T item = (T) items[0];

        for (int i = 1; i < count; i++) {
            items[i - 1] = items[i];
        }
        count--;

        return item;
    }
}
```

这段代码主要有俩问题，一个是这些`while`循环看着很不爽；另一个是在取出元素后逐个将元素左移的操作效率非常的低；而且满天飞的`wait()`和`notifyAll()`也会影响代码的可读性。

那么接下来看看这段代码怎么优化成接近Java自己的实现。

## 优化阻塞

BlockingQueue的一个机制是，在队列空时取元素的操作会被阻塞，而在队列满时放元素的操作会被阻塞。在Java的`ArrayBlockingQueue`中，它并没有使用`synchronized`加锁，而是用了`ReentrantLock`对象。此外对于阻塞线程和唤醒线程操作，它也没有用`wait()`和`notifyAll()`，而是通过两个`Condition`对象`notEmpty`和`notFull`实现。

所以改进方案就是，去掉方法定义中的`synchronized`关键字，换成用`ReentrantLock`加锁解锁；去掉`wait()`和`notifyAll()`，改成用`notEmpty`和`notFull`管理状态以及负责唤醒，同时可以借助这两个变量的语义增强代码的可读性。

```java
public class BlockingQueue<T> {
    private final Object[] items;
    private final int maxSize;
    private int count = 0;

    private final ReentrantLock lock;
    private final Condition notEmpty;
    private final Condition notFull;

    public BlockingQueue(final int size) {
        this.items = new Object[size];
        this.maxSize = size;

        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.notFull = lock.newCondition();
    }

    public void put(final T item) throws InterruptedException {
        // 尝试上锁，使其线程安全
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();

        try {
            while (count >= maxSize) {
                // 在队列满时使当前线程进入等待状态
                // 直到其signal()方法被调用
                notFull.await();
            }

            // 存入元素，更新元素数量
            items[count] = item;
            count++;

            // 通知此时队列非空，唤醒在调用take()方法时因队列空而被阻塞的线程
            notEmpty.signal();
        } finally {
            // 最后无论如何要保证锁被释放
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        // 尝试上锁，使其线程安全
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();

        try {
            while (count == 0) {
                // 在队列空时使当前线程进入等待状态
                // 直到其signal()方法被调用
                notEmpty.await();
            }

            // 取出最早被放入的元素
            final T item = (T) items[0];
            // 领头的元素出列了，后面的往前挪一个位置
            for (int i = 1; i < count; i++) {
                items[i - 1] = items[i];
            }
            // 更新元素数量
            count--;

            // 通知此时队列非满，唤醒在调用put()方法时因队列满而被阻塞的线程
            notFull.signal();

            return item;
        } finally {
            // 最后无论如何要保证锁被释放
            notFull.signal();
        }

    }
}
```

## 优化取出元素后更新队列内容

如果这是一个非常大的队列，那么用一个for循环将所有元素往左移来更新队列内容的方式会消耗巨量的时间，显然它的效率是非常低的。相比于每次取出元素后都重排数组中的元素，Java中`ArrayBlockingQueue`则是利用循环数组的思路，通过两个指针来指示当前该从什么位置取或该向什么位置存（这不是经典的双指针玩法么，前段时间刷LeetCode还做过呢，咋就没想起来），这样每次存取元素后，只需要更新指针指向的位置就行，效率可想而知非常高。

```java
public class BlockingQueue<T> {
    private final Object[] items;
    private int count = 0;
    private int takeIndex = 0;
    private int putIndex = 0;

    private final ReentrantLock lock;
    private final Condition notEmpty;
    private final Condition notFull;

    public BlockingQueue(final int size) {
        this.items = new Object[size];

        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.notFull = lock.newCondition();
    }

    public void put(final T item) throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();

        try {
            while (count >= items.length) {
                notFull.await();
            }

            items[putIndex] = item;
            count++;

            // 存完元素之后，写指针往后挪一位
            // 如果指针挪完之后突破了数组末尾，那么写指针循环到数组头
            // 因为在队列满的时候线程会被阻塞，所以不会出现元素被覆盖的情况
            if (++putIndex == items.length) {
                putIndex = 0;
            }

            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();

        try {
            while (count == 0) {
                notEmpty.await();
            }

            final T item = (T) items[takeIndex];
            items[takeIndex] = null;
            count--;

            // 取出当前元素后，读指针往后挪一位
            // 如果指针挪完之后突破了数组末尾，那么读指针循环到数组头
            // 因为在队列空的时候线程会被阻塞，所以不会出现读到无效元素的情况
            if (++takeIndex == items.length) {
                takeIndex = 0;
            }

            notFull.signal();

            return item;
        } finally {
            notFull.signal();
        }
    }
}
```
