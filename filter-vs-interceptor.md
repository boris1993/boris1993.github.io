---
title: "Filter与interceptor的区别"
source: https://www.boris1993.com/filter-vs-interceptor.html
date: 2022-12-13
updated: 2026-10-10
tags: [Java, Servlet, Spring MVC, Filter, Interceptor]
categories: [学知识]
---

# Filter与interceptor的区别

## Filter

-   `Filter`是`Servlet`规范制定的，受`Servlet`容器支持，接口定义在`javax.servlet`包中。
-   `Filter`是被Web Server（如`Tomcat`）调用。
-   `Filter` 需要在 `web.xml` 中定义之后才会起作用。
-   `Filter`只在请求的前后起作用，`Servlet`对于`Filter`来说是一个黑盒。
-   `Filter`的执行顺序是：
    -   `void init(FilterConfig)` - 容器在初始化`Filter`时调用，在`Filter`生命周期内仅会被调用一次。方法可以抛出`ServletException`。
    -   `doFilter(ServletRequest, ServletResponse, FilterChain)` - Web容器每一次请求都会调用该方法。该方法将容器的请求和响应作为参数传递进来， `FilterChain`用来调用下一个`Filter`。
    -   `void destroy()` - 当容器销毁 Filter 实例时调用该方法，可以在方法中销毁资源，该方法在 Filter 的生命周期只会被调用一次。

## Interceptor

-   `Interceptor`是`Spring`容器内的，由`Spring`框架提供支持。接口`HandlerInterceptor`定义在`org.springframework.web.servlet`包中。
-   `Interceptor`是被`Spring`调用。
-   `Interceptor`可以深入到方法前后、异常抛出前后等，比起`Filter`有更大的弹性。
-   `Interceptor`还允许用户介入请求的生命周期，可以在请求过程中获取信息，通常与请求更加耦合。
-   `Interceptor`的执行顺序是：
    -   请求到达`DispatcherServlet`
    -   `DispatcherServlet`将请求发送至`Interceptor`，`Interceptor`执行`preHandle()`方法
    -   请求到达`Controller`
    -   请求结束后，`Interceptor`执行`postHandle()`方法

## 参考文章

-   Spring Interceptor vs Filter 拦截器和过滤器区别
-   Spring Interceptor vs Filter - 掘金
-   Chapter 10. Filters and Interceptors - Jersey 2.37 User Guide
