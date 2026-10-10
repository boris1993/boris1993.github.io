---
title: "在Java中如何实现在多个线程全部完成后再执行后续的代码"
source: https://www.boris1993.com/java-how-to-wait-for-threads-before-continuing.html
date: 2023-04-05
updated: 2026-10-10
tags: [Java, 多线程, 面试, CountDownLatch, Future]
categories: [学知识]
---

# 在Java中如何实现在多个线程全部完成后再执行后续的代码

面试的时候被问到，如何在5个线程全部执行完毕之后，再继续执行后续的代码。迫于对多线程的东西了解不多，只答出一个`CountDownLatch`，还大概答出一个用`Future`的思路。回来痛定思痛，请出了万能的ChatGPT，学到了其他的几种方法。

## CountDownLatch

在 Java 中可以使用`CountDownLatch`来实现线程间的协调和等待。`CountDownLatch`是一个同步工具类，它允许一个或多个线程等待一组事件发生。

在这个问题中，我们可以创建一个初始值为5的`CountDownLatch`，每个线程完成时调用`countDown()`方法将计数器减一，主线程调用`await()`方法等待计数器归零后再执行后续代码。

下面是一个示例代码：

```java
import java.util.concurrent.CountDownLatch;

public class Main {

    public static void main(String[] args) throws InterruptedException {
        int numThreads = 5;
        CountDownLatch latch = new CountDownLatch(numThreads);
        
        // 创建 5 个线程
        for (int i = 0; i < numThreads; i++) {
            Thread thread = new MyThread(latch);
            thread.start();
        }
        
        // 等待所有线程执行完毕
        // P.S.：我当时想不起来可以await了，就答了个用循环检查CountDownLatch......
        latch.await();
        
        // 执行后续代码
        System.out.println("All threads have finished executing.");
    }

    private static class MyThread extends Thread {
        private final CountDownLatch latch;

        public MyThread(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                // 线程执行一些操作
                // ...
                System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
            } finally {
                // 线程执行完毕后调用 countDown() 方法
                latch.countDown();
            }
        }
    }
}
```

在上面的示例代码中，`MyThread`类表示线程的实现。在`run()`方法中，线程执行一些操作，然后调用`countDown()`方法通知`CountDownLatch`计数器减一。在主线程中，我们创建5个线程并启动它们，然后调用`await()`方法等待所有线程执行完毕。最后，当所有线程执行完毕时，主线程输出一条消息，并继续执行后续代码。

## 使用join()方法

如果不使用`CountDownLatch`，可以使用Java的线程`join()`方法来等待所有线程执行完毕。`join()`方法可以使得一个线程在另一个线程结束后再执行。具体来说，可以在主线程中依次调用每个线程的`join()`方法，让主线程等待每个线程执行完毕后再继续执行后续代码。

下面是一个使用`join()`方法实现的示例代码：

```java
import java.util.ArrayList;
import java.util.List;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        List<MyThread> threads = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            MyThread thread = new MyThread();
            thread.start();
            threads.add(thread);
        }

        for (MyThread thread : threads) {
            // join()方法的JavaDoc写的清晰明了：
            // Waits for this thread to die.
            thread.join();
        }

        System.out.println("All threads have finished executing.");
    }

    private static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
        }
    }
}
```

在上面的示例代码中，`MyThread`类表示线程的实现。在主线程中，我们创建5个线程并启动它们，然后依次调用每个线程的`join()`方法等待线程执行完毕。最后，当所有线程执行完毕时，主线程输出一条消息，并继续执行后续代码。

需要注意的是，`join()`方法会阻塞当前线程，直到被等待的线程执行完毕。因此，在使用`join()`方法时要小心，以避免出现死锁等问题。

## Future

使用`Future`类也可以实现等待多个线程执行完毕。`Future`是Java提供的一种异步计算的机制，可以在一个线程中调用另一个线程并等待其执行结果。具体来说，可以使用 `ExecutorService`的`invokeAll()`方法启动多个线程，将返回的`Future`对象保存到一个列表中，然后调用每个`Future`对象的`get()`方法等待线程执行完毕。

下面是一个使用`Future`类实现的示例代码：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executor = Executors.newFixedThreadPool(5);
        List<Callable<Void>> tasks = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            Callable<Void> task = new MyThread();
            tasks.add(task);
        }

        List<Future<Void>> futures = executor.invokeAll(tasks);

        for (Future<Void> future : futures) {
            // Waits if necessary for the computation to complete,
            // and then retrieves its result.
            future.get();
        }

        // 执行后续代码
        System.out.println("All threads have finished executing.");

        // 关闭线程池
        executor.shutdown();
    }

    private static class MyThread implements Callable<Void> {

        @Override
        public Void call() {
            System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
            return null;
        }
    }
}
```

在上面的示例代码中，`MyTask`类表示线程的实现。在主线程中，我们使用`ExecutorService`的`invokeAll()`方法启动多个线程，并将返回的`Future`对象保存到一个列表中。然后，我们依次调用每个`Future`对象的`get()`方法等待线程执行完毕。最后，当所有线程执行完毕时，主线程输出一条消息，并继续执行后续代码。

需要注意的是，在使用`Future`类时要小心，以避免出现线程池满载等问题。在实际应用中，可以根据需要调整线程池大小或使用其他调度机制来处理大量的并发任务。

## wait()和notifyAll()

可以使用Java的`wait()`和`notifyAll()`方法来实现等待多个线程执行完毕。具体来说，可以在主线程中创建一个共享的计数器变量，每个线程在执行完毕后将计数器减一。当计数器为0时，说明所有线程执行完毕，可以调用`notifyAll()`方法唤醒主线程继续执行后续代码。

下面是一个使用`wait()`和`notifyAll()`方法实现的示例代码：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(5);

        for (int i = 0; i < 5; i++) {
            new MyThread(countDownLatch).start();
        }

        synchronized (countDownLatch) {
            while (countDownLatch.getCount() > 0) {
                countDownLatch.wait();
            }
        }

        System.out.println("All threads have finished executing.");
    }

    private static class MyThread extends Thread {
        private final CountDownLatch countDownLatch;

        public MyThread(CountDownLatch countDownLatch) {
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            try {
                System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
            } finally {
                synchronized (countDownLatch) {
                    countDownLatch.countDown();
                    countDownLatch.notifyAll();
                }
            }

        }
    }
}
```

在上面的示例代码中，MyThread 类表示线程的实现。在主线程中，我们创建 5 个线程并启动它们，然后使用一个共享的计数器变量 countDownLatch 记录线程执行的状态。当每个线程执行完毕时，将计数器减一，并调用 notifyAll() 方法唤醒主线程。在主线程中，我们使用 wait() 方法等待所有线程执行完毕，直到计数器为 0。

P.S.：感觉这个实现有种莫名其妙的别扭感……

## CompletionService

使用`CompletionService`可以比较方便地实现等待所有线程执行完毕的功能。`CompletionService`是Java提供的一个接口，它可以将任务提交给线程池执行，并在任务执行完毕后立即返回结果，从而实现异步执行和结果收集的功能。

具体来说，可以创建一个`ExecutorService`对象作为线程池，然后将任务提交给`CompletionService`执行。在提交任务时，可以使用`submit()`方法返回一个`Future`对象，用于后续获取任务执行的结果。使用`CompletionService`的`take()`方法可以等待任意一个任务执行完毕并返回结果，从而避免了使用`join()`方法等待所有线程执行完毕的阻塞等待。

下面是一个使用`CompletionService`实现的示例代码：

```java
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(5);
        CompletionService<Void> completionService = new ExecutorCompletionService<>(executor);

        for (int i = 0; i < 5; i++) {
            completionService.submit(new MyTask());
        }

        for (int i = 0; i < 5; i++) {
            // Retrieves and removes the Future representing the next completed task,
            // waiting if none are yet present.
            completionService.take();
            System.out.println("Thread " + i + " has finished executing");
        }

        executor.shutdown();

        System.out.println("All threads have finished executing.");
    }

    private static class MyTask implements Callable<Void> {
        @Override
        public Void call() {
            System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
            return null;
        }
    }
}
```

## 线程的getState()方法

除了`CountDownLatch`和`Future`类，还有其他实现方法。其中一个比较简单的方法是使用 Java 的线程状态`（Thread.State）`来判断所有线程是否执行完毕。

具体来说，可以将所有线程保存到一个列表中，然后在主线程中依次调用每个线程的`getState()`方法，检查线程状态是否为`Terminated`。如果所有线程都已经执行完毕，则可以继续执行后续代码。

下面是一个使用线程状态实现的示例代码：

```java
import java.util.ArrayList;
import java.util.List;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        List<MyThread> threads = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            MyThread thread = new MyThread();
            thread.start();
            threads.add(thread);
        }

        boolean allThreadsFinished = false;
        while (!allThreadsFinished) {
            allThreadsFinished = true;

            for (MyThread thread : threads) {
                if (thread.getState() != Thread.State.TERMINATED) {
                    allThreadsFinished = false;
                    break;
                }
            }

            Thread.sleep(100);
        }

        System.out.println("All threads have finished executing.");
    }

    private static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
        }
    }
}
```

在上面的示例代码中，`MyThread`类表示线程的实现。在主线程中，我们创建5个线程并启动它们，然后循环检查每个线程的状态，直到所有线程都执行完毕。在每次循环中，我们先将`allThreadsFinished`标志设为`true`，然后依次检查每个线程的状态。如果有任何一个线程的状态不是`Terminated`，则将`allThreadsFinished`标志设为`false`，并跳出循环。等待一段时间后重新检查线程状态，直到所有线程都执行完毕。

需要注意的是，使用线程状态进行等待需要定期检查所有线程的状态，因此会占用一定的CPU资源。在实际应用中，可以根据需要调整等待的时间间隔以及检查的次数，以平衡等待时间和CPU资源的消耗。

## CyclicBarrier

使用`CyclicBarrier`也可以比较方便地实现等待所有线程执行完毕的功能。`CyclicBarrier`是Java提供的一个同步辅助类，它可以让一组线程等待彼此达到某个共同的屏障点。

具体来说，可以创建一个`CyclicBarrier`对象，并指定需要等待的线程数量。每个线程在执行完自己的任务后，调用`CyclicBarrier`的`await()`方法，表示已经到达了屏障点。当所有线程都到达了屏障点后，`CyclicBarrier`就会释放所有线程，从而实现等待所有线程执行完毕的功能。

下面是一个使用`CyclicBarrier`实现的示例代码：

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        // 创建 CyclicBarrier 对象，等待 5 个线程
        final CyclicBarrier cyclicBarrier =
                new CyclicBarrier(
                        5,
                        () -> {
                            // 所有线程到达屏障点时执行的操作
                            System.out.println("All threads have finished executing.");
                        });

        for (int i = 0; i < 5; i++) {
            new Thread(new MyTask(cyclicBarrier)).start();
        }
    }

    private static class MyTask implements Runnable {
        private final CyclicBarrier cyclicBarrier;

        public MyTask(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            // 等待所有线程执行完毕
            try {
                System.out.println("Thread " + Thread.currentThread().getId() + " has finished executing.");
                cyclicBarrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

在上面的示例代码中，`MyTask`类表示线程的实现。在主线程中，我们创建一个`CyclicBarrier`对象，并指定需要等待的线程数量为5。每个线程在执行完自己的任务后，调用`CyclicBarrier`的`await()`方法，表示已经到达了屏障点。当所有线程都到达了屏障点后，`CyclicBarrier`就会执行屏障操作，这里是输出`All threads have finished executing.`。

需要注意的是，如果其中一个线程在等待过程中被中断或者抛出异常，那么`CyclicBarrier`就会被破坏，所有线程都会被唤醒并抛出`BrokenBarrierException`异常。因此，在实现时需要捕获`InterruptedException`和`BrokenBarrierException`异常。
