---
title: "循序渐进写一个Servlet(5) - Filter"
source: https://www.boris1993.com/java-servlet-5-filter.html
date: 2019-03-13
updated: 2026-10-10
tags: [Java, Servlet, Filter]
categories: [学知识]
---

# 循序渐进写一个Servlet(5) - Filter

Servlet（Server Applet），全称Java Servlet，是用Java编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态Web内容。本系列将一步步地写出一个Servlet程序。

这篇博文将演示如何创建和使用filter。

# 什么是`Filter`

当客户端向servlet容器发送请求时，请求通常会直接发送到servlet进行处理，就像下图这样：

![Request flow without Filter](https://blog-static.boris1993.com/java-servlet-5-filter/request-flow-without-filter.png)

但是，如果希望在请求被servlet处理之前和之后，再进行一些附加的处理，就可以使用`Filter`完成。

![Request flow with Filter](https://blog-static.boris1993.com/java-servlet-5-filter/request-flow-with-filter.png)

一个常见的使用场景是，在filter中定义如何检查请求是否合法，比如请求头中是否携带了有效的认证和鉴权信息；或者可以在filter中针对请求和响应记录日志。

# 怎么使用`Filter`

`javax.servlet.Filter`接口定义了一个filter的生命周期，要创建一个filter，就要实现`Filter`接口。

`Filter`接口包含下列方法声明：

1.  `init()`，用于定义在初始化这个filter时要执行的操作，该方法在filter的生命周期内只会执行一次；
2.  `doFilter()`，用于定义这个filter要进行的操作，每当有请求被发送到与该filter绑定的资源时，该方法都会被执行一次；
3.  `destroy()`，用于定义在停止这个filter时要执行的操作，只会在一个filter被销毁时执行。

## 创建一个实现`Filter`接口的类

```java
public class Filter1 implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

    }

    @Override
    public void destroy() {

    }
}
```

## 定义这个filter的行为

在`doFilter()`方法内定义这个filter的行为。

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

    // 在处理请求前打印信息
    System.out.println("Request passing through Filter 1");

    // 交由FilterChain将请求交给下一个filter或交给servlet处理
    chain.doFilter(request, response);

    // servlet发送响应后打印信息
    System.out.println("Response passing througe Filter 1");
}
```

## 在容器中注册filter

与servlet一样，filter也需要在容器中注册之后才能发挥作用。注册filter也有两种方式：通过`web.xml`，或者通过`@WebFilter`注解。

这里有一点需要注意，虽然filter之间没有依赖关系，但是如果要保证filter的执行顺序，那么必须使用`web.xml`来注册。

Servlet 3.0 规范的`8.2.3`节中有如下说明：

> If the order in which the listeners, servlets, filters are invoked is important to an application then a deployment descriptor must be used.

因为使用注解注册的filter，其调用顺序没有在规范中指定。

> As described above, when using annotations to define the listeners, servlets and filters, the order in which they are invoked is unspecified.

如果一定要使用注解并保证filter的执行顺序，那么可以参考Stack Overflow中这篇回答。

为了演示filter的执行顺序，这里再增加一个名为`Filter2`的filter，内容与`Filter1`类似。

### `web.xml`

在`web.xml`中增加如下配置：

```xml
<filter>
    <filter-name>filter1</filter-name>
    <filter-class>com.boris.tomcatlistener.filter.Filter1</filter-class>
</filter>
<filter-mapping>
    <filter-name>filter1</filter-name>
    <url-pattern>/demoServlet</url-pattern>
</filter-mapping>

<filter>
    <filter-name>filter2</filter-name>
    <filter-class>com.boris.tomcatlistener.filter.Filter2</filter-class>
</filter>
<filter-mapping>
    <filter-name>filter2</filter-name>
    <url-pattern>/demoServlet</url-pattern>
</filter-mapping>
```

`filter`标签描述了一个filter的基本信息，其中filter名称(`filter-name`)和filter所在类(`filter-class`)为必填项。

`filter-mapping`标签描述了一个filter将与哪个URL或者与哪个servlet绑定，`filter-name`指定使用哪个filter处理请求，`url-pattern`指定发往哪个URL的请求会触发这个filter，`servlet-name`指定发往哪个servlet的请求会触发这个filter。`url-pattern`和`servlet-name`可以同时存在，也可以同时存在多个。

`filter-mapping`标签的先后顺序，将决定filter链中各个filter被调用的先后顺序。如上文中先配置了`filter1`后配置了`filter2`，那么在请求到达时，会先执行`filter1`然后再执行`filter2`。

配置完毕后部署并运行该项目，向`http://localhost:8080/servletdemo/DemoServlet`发送一个请求，在控制台可以看到如下输出：

```text
Request passing through Filter 1
Request passing through Filter 2
Response passing througe Filter 2
Response passing througe Filter 1
```

### `@WebFilter`注解

`@WebFilter`是`Servlet 3.0`中新增的特性，在`Tomcat 7`及以前版本中将无法工作。

以`Filter1`为例，为其添加如下注解：

```java
@WebFilter(
    filterName = "filter1",
    urlPatterns = "/demoServlet"
)
```

`filter-name`属性指定了这个filter的名称。

有三个属性可以指定filter的触发条件：

1.  `value`
2.  `urlPatterns`
3.  `servletNames`

以上三个属性都可以接受一个字符串，或者用大括号包括起来的多个字符串。

在注解只有一个参数，并且该参数是指定要匹配的URL时，建议使用`value`属性，比如这样：

```java
// value为默认的属性
@WebFilter("/demoServlet")

// 显式指定value属性
@WebFilter(value = "/demoServlet")
```

否则，建议使用`urlPatterns`属性和`servletNames`属性。不允许`value`和`urlPatterns`同时出现。

Servlet 3.0 规范的`8.1.2 @WebFilter`节中说明原文如下：

> It is recommended to use value when the only attribute on the annotation is the url pattern and to use the urlPatterns attribute when the other attributes are also used. It is illegal to have both value and urlPatterns attribute used together on the same annotation.

配置完毕后部署并运行该项目，向`http://localhost:8080/servletdemo/DemoServlet`发送一个请求，在控制台可以看到如下输出：

```text
Request passing through Filter 1
Response passing througe Filter 1
```

# 系列博文

-   [循序渐进写一个Servlet(1) - 介绍相关的接口和类](https://www.boris1993.com/projects/java/Servlet/java-servlet-1-introducing-classes-and-interfaces.html)
-   [循序渐进写一个Servlet(2) - 第一个servlet](https://www.boris1993.com/projects/java/Servlet/java-servlet-2-first-servlet.html)
-   [循序渐进写一个Servlet(3) - 分别处理GET和POST](https://www.boris1993.com/projects/java/Servlet/java-servlet-3-handle-get-and-post.html)
-   [循序渐进写一个Servlet(4) - 会话追踪](https://www.boris1993.com/projects/java/Servlet/java-servlet-4-session-tracking.html)
-   [循序渐进写一个Servlet(5) - Filter](https://www.boris1993.com/projects/java/Servlet/java-servlet-5-filter.html)
