---
title: "Servlet和Spring MVC是不是线程安全的"
source: https://www.boris1993.com/are-servlet-and-spring-mvc-thread-safe.html
date: 2022-12-14
updated: 2026-10-10
tags: [Java, 多线程, 线程安全]
categories: [学知识]
---

# Servlet和Spring MVC是不是线程安全的

太长不看：默认情况下`servlet`不是线程安全的；`Spring bean`是否线程安全取决于这个bean的scope。

## Servlet为什么不是线程安全的

单个servlet实例中的方法会被多个线程同时调用很多次，而servlet类中的变量是引用传递，多个线程同时存取某个变量时，就会产生线程安全问题。

以下方法可以让servlet或其中的一部分代码块变成线程安全的：

-   使用`syncronized`关键字。

```java
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class MyServlet extends HttpServlet {

    public void doGet (HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        //pre-processing
        synchronized (this) {
            //code in this block is thread-safe
        }
        //other processing;
    }
}
```

-   使用`SingleThreadModel`类

如果在`Sun Java System Web Server 7.0`中部署单线程servlet时，servlet引擎会创建一个servlet实例池，并保持内存中有多个servlet实例的副本。要想调整实例池中的实例数量，可以调整Web Server中的`singleThreadedServletPoolSize`属性。这种情况下，服务器的性能可能会降低，如果池中的所有实例全部被占用，那么新到来的请求就必须要在队列中等待某个实例被释放。

```java
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class myServlet extends HttpServlet implements SingleThreadModel {
    //servlet methods...
}
```

## Spring Bean什么时候是线程安全的，什么时候不是

首先，`singleton`的bean不是线程安全的。在不使用`@Lazy`注解时，框架会在启动时就初始化好singleton bean。但是框架不会管理开发人员怎么用这些singleton bean，换言之，开发人员要自己处理线程安全问题。

而被`@RequestScope`注解标记的bean是线程安全的，因为这些bean不会在线程之间共享，而是在每次请求到来时都会创建新的实例。

Session scope的bean也不是线程安全的，因为这些bean会与用户的会话绑定，每有一个新用户发来请求，就会创建一个新的bean。但是，用户发来的请求可能是并行的，所以也有可能产生线程安全问题。

Prototype bean是不是线程安全取决于它与哪种bean绑定。如果它是被一个singleton bean所依赖，因为singleton bean不是线程安全的，这个prototype bean也将被多个线程访问，所以此时prototype bean不是线程安全的；如果它被一个request scope的bean所依赖，那么这个prototype bean就是线程安全的，因为它会随着request scoped bean消亡而消亡，同时不会被多个线程共享。

除此之外，如果一个bean是无状态的，那么无论是什么scope，它都是线程安全的。

## 参考文档

-   Handling Threading Issues
-   Spring bean thread safety guide
