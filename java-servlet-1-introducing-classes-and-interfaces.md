---
title: "循序渐进写一个Servlet(1) - 介绍相关的接口和类"
source: https://www.boris1993.com/java-servlet-1-introducing-classes-and-interfaces.html
date: 2019-03-06
updated: 2026-10-10
tags: [Java, Servlet]
categories: [学知识]
---

# 循序渐进写一个Servlet(1) - 介绍相关的接口和类

Servlet（Server Applet），全称Java Servlet，是用Java编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态Web内容。本系列将一步步地写出一个Servlet程序。

这篇博文将介绍一些后面会用到的接口和类。

# 常用的类和接口

## `javax.servlet.Servlet`

`javax.servlet.Servlet`是一个接口，它定义了一个servlet必须要实现的方法，包括如何初始化一个servlet，如何让这个servlet处理请求，以及如何将这个servlet从容器中移除，即“生命周期方法”。它们被调用的顺序，即servlet的生命周期，是这样的：

1.  容器生成这个servlet的对象，然后调用其`init()`方法完成初始化步骤
2.  当收到请求之后，会调用其`service()`方法来处理请求
3.  当这个servlet停止服务时，`destroy()`方法会被调用并准备销毁

Servlet的启动时间由`web.xml`中`load-on-startup`属性决定，当值为负数或未设定该属性时，容器可以自由决定何时初始化该servlet；当值为零或正整数时，容器需要在启动时就初始化该servlet，此时该属性的值代表初始化的优先级，值越小优先级越高，对于有相同优先级的servlet，容器可以自行决定加载顺序。

当应用被重新部署，或者在容器停机时，servlet会被销毁，同时servet无法被手动销毁。

除了生命周期方法外，`Servlet`接口还有两个方法：

1.  `getServletConfig()`可以让servlet取到启动相关的信息
2.  `getServletInfo()`用来获取servlet相关的信息，如作者、版本号、版权信息等

## `javax.servlet.GenericServlet`

`GenericServlet`类是一个抽象类，定义了一个协议无关的，通用的servlet实现。

它预置了`ServletConfig`接口的实现，并且简易实现了`Servlet`接口中的`init()`方法和`destroy()`方法，开发人员只需要覆盖`service()`方法。

## `javax.servlet.http.HttpServlet`

`HttpServlet`类提供了一个用于处理HTTP请求的servlet基类。一个`HttpServlet`的子类至少需要覆盖一个方法，而且通常是覆盖如下几个方法：

1.  `doGet()`，以实现对`HTTP GET`请求的处理
2.  `doPost()`，以实现对`HTTP POST`请求的处理
3.  `doPut()`，以实现对`HTTP PUT`请求的处理
4.  `doDelete()`，以实现对`HTTP DELETE`请求的处理
5.  `init()`和`destroy()`，以管理servlet生命周期内所需的资源
6.  `getServletInfo()`，可以用来自定义servlet返回哪些关于自身的信息

此外，不建议直接覆盖`service()`方法，因为`HttpServlet#service()`方法中已经实现了根据HTTP请求类型调用对应的`doXXX()`方法。

如果某个HTTP方法对应的`doXXX()`方法没有被覆盖，则视为该servlet不支持这个HTTP方法。如在没有覆盖`doGet()`时收到`HTTP GET`请求，则会返回`HTTP 405 METHOD NOT ALLOWED (对应HTTP 1.1)`错误码，或`HTTP 400 BAD REQUEST (对应其他HTTP版本)`错误码。

需要注意的是，`HttpServlet`类并没有实现`HTTP CONNECT`和`HTTP PATCH`方法。当请求这两个方法，或其他非标准方法时，将会返回`HTTP 501 NOT IMPLEMENTED`错误码。

# 常用的常量

-   HTTP方法相关的常量位于`HttpServlet`类中，是String类型，命名规则为`METHOD_方法`，如`METHOD_GET`
-   HTTP状态码相关的常量位于`HttpServletResponse`类中，是int类型，命名规则为`SC_状态码名`，如`SC_OK`、`SC_NOT_FOUND`

# 系列博文

-   [循序渐进写一个Servlet(1) - 介绍相关的接口和类](https://www.boris1993.com/projects/java/Servlet/java-servlet-1-introducing-classes-and-interfaces.html)
-   [循序渐进写一个Servlet(2) - 第一个servlet](https://www.boris1993.com/projects/java/Servlet/java-servlet-2-first-servlet.html)
-   [循序渐进写一个Servlet(3) - 分别处理GET和POST](https://www.boris1993.com/projects/java/Servlet/java-servlet-3-handle-get-and-post.html)
-   [循序渐进写一个Servlet(4) - 会话追踪](https://www.boris1993.com/projects/java/Servlet/java-servlet-4-session-tracking.html)
-   [循序渐进写一个Servlet(5) - Filter](https://www.boris1993.com/projects/java/Servlet/java-servlet-5-filter.html)
