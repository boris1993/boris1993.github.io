---
title: "从字节码看synchronized关键字是怎么工作的"
source: https://www.boris1993.com/how-synchronized-works-in-java.html
date: 2023-04-05
updated: 2026-10-10
tags: [Java, 面试问题, syncronized]
categories: [学知识]
---

# 从字节码看synchronized关键字是怎么工作的

昨天面试的时候被问到Java中的`synchronized`关键字是什么原理，虽然凭着记忆打出来是通过控制对象头的Monitor来实现，但是毕竟没吃透这个知识点，还是没啥底气。干脆，这次就从字节码上看看，用了`synchronized`关键字的方法，到底是怎么执行的。

## 示例代码

说起`synchronized`的最简单的使用场景，我马上就想起双检单例模式。

```java
public class Test {
    private static volatile Test INSTANCE;

    private Test() {
    }

    public static Test getInstance() {
        if (INSTANCE == null) {
            synchronized (Test.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Test();
                }
            }
        }

        return INSTANCE;
    }

    public void print() {
        System.out.println("test");
    }
}
```

## 反编译成字节码

把`Test`类先编译了，然后用`javap -c Test.class`反编译，就能看到这个类的字节码了。

```java
Compiled from "Test.java"
public class Test {
  public static Test getInstance();
    Code:
       0: getstatic     #7                  // 把静态变量INSTANCE加载到栈
       3: ifnonnull     37                  // 如果值不是null，那么跳转到标签37
       6: ldc           #8
       8: dup
       9: astore_0
      10: monitorenter                      // 进入synchronized块
      11: getstatic     #7                  // 把静态变量INSTANCE加载到栈
      14: ifnonnull     27                  // 如果值不是null，那么跳转到标签27
      17: new           #8                  // new一个Test对象
      20: dup
      21: invokespecial #13                 // 执行构造函数
      24: putstatic     #7
      27: aload_0
      28: monitorexit                       // 退出synchronized块
      29: goto          37
      32: astore_1
      33: aload_0
      34: monitorexit
      35: aload_1
      36: athrow
      37: getstatic     #7                  // Field INSTANCE:LTest;
      40: areturn
    Exception table:
       from    to  target type
          11    29    32   any
          32    35    32   any

  public void print();
    Code:
       0: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #20                 // String test
       5: invokevirtual #22                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

注意看`10: monitorenter`和`28: monitorexit`这两条字节码，这就是`synchronized`关键字实际做了的事。

## Java对象头和Monitor

要说明白`monitorenter`和`monitorexit`实际干了点啥，那就得先整明白Java对象的对象头。

一个Java对象，在内存中的布局包括三块区域：对象头、实例数据、和对齐填充。

![](https://blog-static.boris1993.com/how-synchronized-works-in-java/java_object_memory_allocation.png)

别的东西咱们先不看，只看对象头这部分。对象头的最后2bit就存储了锁的标志位。

至于Monitor，Java官方文档是这么描述的：

> Synchronization is built around an internal entity known as the intrinsic lock or monitor lock. (The API specification often refers to this entity simply as a “monitor.”) Intrinsic locks play a role in both aspects of synchronization: enforcing exclusive access to an object’s state and establishing happens-before relationships that are essential to visibility.
> 
> Every object has an intrinsic lock associated with it. By convention, a thread that needs exclusive and consistent access to an object’s fields has to acquire the object’s intrinsic lock before accessing them, and then release the intrinsic lock when it’s done with them.
> 
> 同步是围绕着一个名为“内在锁”或“monitor锁”的机制构建的。（API规范文档中，通常会称其为“monitor”）  
> 内在锁一方面保证了针对一个对象的专属访问权限，另一方面保证了对可见性很重要的happens-before原则。  
> 每个对象都会有一个与其相关联的内在锁。按照约定，如果一个线程需要持续持有对一个对象的独家访问权限，那么这个线程必须先获得到这个对象的内在锁，然后在执行完毕后释放掉这个内在锁。

代码执行到`monitorenter`指令，说明开始进入`synchronized`代码块，这时候JVM会尝试获取这个对象的`monitor`所有权，即尝试加锁；而执行到`monitorexit`指令，就说明要么`synchronized`代码块执行完毕，要么代码执行的时候抛出了异常，这时候JVM就会释放这个对象的`monitor`所有权，即释放锁。

## 继续深入细节

上面说的也是云里雾里的，咱继续往深处挖，看看具体的实现。

`Monitor`这个东西，看Java源码找不到，得找虚拟机的C++源码。比如我们常用的HotSpot虚拟机中，`Monitor`是由`ObjectMonitor`类实现的：

```c++
// 为了解释方便，仅抄录了相关的代码，并重排了位置
class ObjectMonitor {
    public:
        ObjectMonitor() {
            _header       = NULL;
            _count        = 0;
            _waiters      = 0,
            _recursions   = 0;
            _object       = NULL;
            _owner        = NULL;
            _WaitSet      = NULL;
            _WaitSetLock  = 0 ;
            _Responsible  = NULL ;
            _succ         = NULL ;
            _cxq          = NULL ;
            FreeNext      = NULL ;
            _EntryList    = NULL ;
            _SpinFreq     = 0 ;
            _SpinClock    = 0 ;
            OwnerIsThread = 0 ;
            _previous_owner_tid = 0;
        }
    
    private:
        volatile intptr_t  _count;          // reference count to prevent reclaimation/deflation
                                            // at stop-the-world time.  See deflate_idle_monitors().
                                            // _count is approximately |_WaitSet| + |_EntryList|
    
    // 等待锁的线程会被封装成ObjectWaiter对象
    protected:
        void *  volatile _owner;                // 一个指针，指向当前拥有锁的线程
        ObjectWaiter * volatile _WaitSet;       // 一个队列，保存着waiting状态的线程
        ObjectWaiter * volatile _EntryList ;    // 一个队列，保存着因等待锁而被阻塞的线程
}
```

当多个线程同时访问一段`synchronized`代码时，会发生这些操作：

-   线程首先会进入`_EntryList`，在该线程获取到对象的`monitor`之后，`_owner`会指向这个线程，然后`_count`计数器加一。
    -   如果得到`monitor`的这个线程调用了`wait()`方法，那么这个线程将会释放掉monitor的所有权，`_owner`变量变回NULL，`_count`计数器也会减一，同时这个线程会进入`_WaitSet`等待被唤醒。
    -   如果这个线程执行完毕，那么它也将释放`monitor`，并复位`_count`的值，这样其他的线程也就可以获得`monitor`来加锁了。
-   上一个线程释放掉`monitor`后，`_EntryList`中的线程就会开始争抢`monitor`，具体哪个线程能成功得到`monitor`是不确定的。

而正因为Monitor对象存在于每个Java对象头的`mark word`中，所以每个Java对象都可以用作锁。

## 参考文章

-   synchronized与对象的Monitor
-   Intrinsic Locks and Synchronization
-   啃碎并发（七）：深入分析Synchronized原理
-   objectMonitor.hpp - JetBrains/jdk8u\_hotspot
-   Why do we need to call ‘monitorexit’ instruction twice when we use ‘synchronized’ keyword? - StackOverflow
