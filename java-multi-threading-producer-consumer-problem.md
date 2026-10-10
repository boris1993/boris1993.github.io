---
title: "生产者与消费者问题在Java中的实现"
source: https://www.boris1993.com/java-multi-threading-producer-consumer-problem.html
date: 2019-03-11
updated: 2026-10-10
tags: [Java, 多线程]
categories: [学知识]
---

# 生产者与消费者问题在Java中的实现

生产者与消费者问题(Producer-consumer problem)，也叫有限缓冲问题(Bounded-buffer problem)，是一个经典的多线程同步问题案例。

该问题中有两个线程共享一个固定大小的缓冲区，一个线程作为生产者，负责向缓冲区中放入数据；另一个线程作为消费者，负责从缓冲区中取出数据。该问题的重点在于，要保证当缓冲区满时，生产者不能继续向其中放入数据，而当缓冲区空时，消费者也不能从缓冲区中取出数据。

那么要保证以上两点，需要在缓冲区空时休眠消费者线程，并当缓冲区有数据之后唤醒消费者线程；以及当缓冲区满时休眠生产者线程，在缓冲区有空闲空间后唤醒生产者线程，或者直接在缓冲区满时放弃未存入缓冲区的数据。

这个问题的难点在于可能会产生死锁。当陷入死锁时，生产者和消费者都会处于休眠状态，并等待对方唤醒自己。

# 使用同步锁实现

## 产品类

产品类代表将要被生产和消费的产品。

```java
/**
 * 生产者-消费者问题 -- 产品类
 */
public class Product {
    private int productId;

    public Product(int productId) {
        this.productId = productId;
    }

    @Override
    public String toString() {
        return "Product{" +
                "productId=" + productId +
                '}';
    }

    public int getProductId() {
        return productId;
    }

    public void setProductId(int productId) {
        this.productId = productId;
    }
}
```

## 仓库类

仓库类用来构造一个存放产品的数组，并带有存取数组的方法(pop/push)，本质上是一个栈。

```java
/**
 * 生产者-消费者问题 -- 仓库类(缓冲区)
 */
public class Warehouse {
    private Product[] products;

    /**
     * 栈顶指针
     */
    private int top = 0;

    public Warehouse() {
        products = new Product[10];
    }

    public Warehouse(int capacity) {
        products = new Product[capacity];
    }

    /**
     * 生产产品
     *
     * @param product 产品
     */
    public synchronized void push(Product product) {

        // 如果仓库已满
        if (top == products.length) {
            try {
                System.out.println("Warehouse full.");

                // 将生产者线程置于等待态
                wait();
            } catch (InterruptedException e) {
                System.out.println("Warehouse full but failed to wait. Reason:");
                System.out.println(e.getMessage());
            }
        }

        products[top++] = product;

        // 现在仓库已有产品
        // 可以唤醒消费者线程
        notifyAll();
    }

    /**
     * 消费产品
     *
     * @return 取出的产品
     */
    public synchronized Product pop() {
        Product product = null;

        // 如果仓库空
        while (products[0] == null) {
            System.out.println("Warehouse empty");

            // 唤醒生产者线程
            notifyAll();

            try {
                // 将消费者线程置于等待态
                wait();
            } catch (InterruptedException e) {
                System.out.println("Warehouse empty but failed to wait. Reason:");
                System.out.println(e.getMessage());
            }
        }

        product = products[--top];
        products[top] = null;

        // 仓库非满，可以唤醒生产者线程
        notifyAll();

        return product;
    }
}
```

## 生产者

```java
public class Producer implements Runnable {

    private String producerName;
    private Warehouse warehouse;
    private Random random = new Random();

    public Producer(String producerName, Warehouse warehouse) {
        this.producerName = producerName;
        this.warehouse = warehouse;
    }

    @Override
    public void run() {
        produce();
    }

    /**
     * 生产产品并存入仓库
     */
    private void produce() {
        int i = 0;

        while (true) {
            Product product = new Product(i++);
            warehouse.push(product);

            System.out.println("[PRODUCED] Product " + product.getProductId());

            try {
                Thread.sleep(random.nextInt(20) * 100);
            } catch (InterruptedException e) {
                System.out.println("Exception occurred in producer thread. Reason: " + e.getMessage());
            }
        }
    }
}
```

## 消费者

```java
public class Consumer implements Runnable {

    private String consumerName;
    private Warehouse warehouse;
    private Random random = new Random();

    public Consumer(String consumerName, Warehouse warehouse) {
        this.consumerName = consumerName;
        this.warehouse = warehouse;
    }

    @Override
    public void run() {
        consume();
    }

    private void consume() {
        while (true) {
            Product product = warehouse.pop();
            System.out.println("[CONSUMED] Product " + product.getProductId());

            try {
                Thread.sleep(random.nextInt(20) * 100);
            } catch (InterruptedException e) {
                System.out.println("Exception occurred in consumer thread. Reason: " + e.getMessage());
            }
        }
    }
}
```

## Main类

```java
public class Main {

    public static void main(String[] args) {
        Warehouse warehouse = new Warehouse();

        Producer producer = new Producer("producer", warehouse);
        Consumer consumer = new Consumer("consumer", warehouse);

        ExecutorService executorService = Executors.newCachedThreadPool();

        executorService.execute(consumer);
        executorService.execute(producer);

        executorService.shutdown();

        while (!executorService.isTerminated()) {
        }

        System.out.println("Thread pool is down");
    }
}
```

## 运行结果

```text
Warehouse empty
[PRODUCED] Product 0
[CONSUMED] Product 0
[PRODUCED] Product 1
[CONSUMED] Product 1
[PRODUCED] Product 2
[PRODUCED] Product 3
[PRODUCED] Product 4
[PRODUCED] Product 5
[CONSUMED] Product 5
[PRODUCED] Product 6
[CONSUMED] Product 6
[CONSUMED] Product 4
[CONSUMED] Product 3
[CONSUMED] Product 2
[PRODUCED] Product 7
[CONSUMED] Product 7
[PRODUCED] Product 8
[CONSUMED] Product 8
Warehouse empty
[PRODUCED] Product 9
[CONSUMED] Product 9
Warehouse empty
```

# 使用阻塞队列实现

相比较于队列，阻塞队列(Blocking queue)可以在队列空时阻塞取值操作，并在队列满时阻塞存入操作。

实际上根据调用不同的方法，可以实现在队列空/满时抛出异常、返回特殊值、阻塞操作、带超时的阻塞操作，具体请参考BlockingQueue文档

## 产品类和仓库类

产品类实现同上，仓库使用阻塞队列(`ArrayBlockingQueue`)实现。

## 生产者

```java
public class Producer implements Runnable {

    private String producerName;
    private BlockingQueue<Product> warehouse;
    private Random random = new Random();

    public Producer(String producerName, BlockingQueue<Product> warehouse) {
        this.producerName = producerName;
        this.warehouse = warehouse;
    }

    @Override
    public void run() {
        produce();
    }

    /**
     * 生产产品并存入仓库
     */
    private void produce() {
        int i = 0;

        while (true) {
            Product product = new Product(i++);
            try {
                warehouse.put(product);
            } catch (InterruptedException e) {
                System.out.println("Exception occurred when putting product in producer. Reason: " + e.getMessage());
            }

            System.out.println("[PRODUCED] Product " + product.getProductId());

            try {
                Thread.sleep(random.nextInt(20) * 100);
            } catch (InterruptedException e) {
                System.out.println("Exception occurred in producer thread. Reason: " + e.getMessage());
            }
        }
    }
}
```

## 消费者

```java
public class Consumer implements Runnable {

    private String consumerName;
    private BlockingQueue<Product> warehouse;
    private Random random = new Random();

    public Consumer(String consumerName, BlockingQueue<Product> warehouse) {
        this.consumerName = consumerName;
        this.warehouse = warehouse;
    }

    @Override
    public void run() {
        consume();
    }

    private void consume() {
        while (true) {
            Product product = null;
            try {
                product = warehouse.take();
            } catch (InterruptedException e) {
                System.out.println("Exception occurred when taking product in consumer. Reason: " + e.getMessage());
            }
            System.out.println("[CONSUMED] Product " + product.getProductId());

            try {
                Thread.sleep(random.nextInt(20) * 100);
            } catch (InterruptedException e) {
                System.out.println("Exception occurred in consumer thread. Reason: " + e.getMessage());
            }
        }
    }
}
```

## 运行结果

```text
[PRODUCED] Product 0
[CONSUMED] Product 0
[PRODUCED] Product 1
[CONSUMED] Product 1
[PRODUCED] Product 2
[CONSUMED] Product 2
[PRODUCED] Product 3
[CONSUMED] Product 3
[PRODUCED] Product 4
[CONSUMED] Product 4
[PRODUCED] Product 5
[PRODUCED] Product 6
[CONSUMED] Product 5
[PRODUCED] Product 7
[PRODUCED] Product 8
[CONSUMED] Product 6
[PRODUCED] Product 9
```
