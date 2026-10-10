---
title: "循序渐进写一个Servlet(2) - 第一个servlet"
source: https://www.boris1993.com/java-servlet-2-first-servlet.html
date: 2019-03-06
updated: 2026-10-10
tags: [Java, Servlet]
categories: [学知识]
---

# 循序渐进写一个Servlet(2) - 第一个servlet

Servlet（Server Applet），全称Java Servlet，是用Java编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态Web内容。本系列将一步步地写出一个Servlet程序。

这篇博文将演示如何创建一个servlet。

# 新建项目

首先，使用Maven的`maven-archetype-webapp`创建一个Java Webapp项目。这样就可以得到一个Java Webapp项目的基本骨架。将项目命名为`servletdemo`。

# 创建Servlet对象

新建一个名为DemoServlet的类。

因为`GenericServlet`已经实现了一个servlet的生命周期方法，而且这里也不需要对生命周期方法做定制化，所以直接继承一个`GenericServlet`就可以。

```java
public class DemoServlet extends GenericServlet {

    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {

    }
}
```

# 实现这个servlet的功能

`service()`方法定义了这个servlet的具体实现，这里先让它返回一个HTML。

```java
@Override
public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {

    // 设定返回内容的MIME类型
    res.setContentType("text/html");

    // 设定内容以UTF-8编码
    res.setCharacterEncoding("utf-8");

    // Java 8 新特性 - try with resources
    // 该特性优点在于，try代码块执行完毕后，会自动close相关资源
    // 要求资源所属的类必须实现了 AutoCloseable 或 Closeable 接口
    try (PrintWriter writer = res.getWriter()) {
        // 开始输出HTML文本
        writer.print("<html lang=\"en\">");
        writer.print("<body>");
        writer.print("<b>Response from DemoServlet</b>");
        writer.print("</body>");
        writer.print("</html>");
    }
}
```

# 注册到容器

光有servlet还不够，下面还需要让容器知道有这个servlet，以及知道要把哪些请求发往这个servlet。这里可以通过修改`web.xml`的方式，也可以通过注解的方式完成注册。

## `web.xml`

`web.xml`位于`src/main/webapp/WEB_INF/web.xml`。

向其中添加如下内容：

```xml
<!-- servlet属性定义了一个servlet的名字和对应的类 -->
<servlet>
    <servlet-name>DemoServlet</servlet-name>
    <servlet-class>com.boris.tomcatlistener.servlet.DemoServlet</servlet-class>
</servlet>

<!-- servlet-mapping属性定义了匹配某个URL的请求应该发往哪个servlet -->
<servlet-mapping>
    <!-- 目标servlet的servlet-name -->
    <servlet-name>DemoServlet</servlet-name>
    <!-- 匹配的URL -->
    <url-pattern>/demoServlet</url-pattern>
</servlet-mapping>
```

## 注解

`WebServlet(String value)`注解表明了这个类是一个servlet，其中`value`的值等同于`web.xml`中`url-patterm`参数。

在项目部署时，容器会处理`WebServlet`注解，并将这个servlet与指定的URL pattern绑定。

```java
@WebServlet(value = "/demoServlet")
public class DemoServlet extends GenericServlet {

    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {

        // 设定返回内容的MIME类型
        res.setContentType("text/html");

        // 设定内容以UTF-8编码
        res.setCharacterEncoding("utf-8");

        // Java 8 新特性 - try with resources
        // 该特性优点在于，try代码块执行完毕后，会自动close相关资源
        // 要求资源所属的类必须实现了 AutoCloseable 或 Closeable 接口
        try (PrintWriter writer = res.getWriter()) {
            // 开始输出HTML文本
            writer.print("<html lang=\"en\">");
            writer.print("<body>");
            writer.print("<b>Response from DemoServlet</b>");
            writer.print("</body>");
            writer.print("</html>");
        }
    }
}
```

# 运行起来吧，servlet哟！

在上面步骤中，创建一个servlet的所有步骤就完成了，尽管这个servlet的功能及其有限，它甚至不能区分发来的HTTP请求。

那么，就将这个servlet部署到容器中，让它开始工作吧。启动成功后，使用浏览器，或者HTTP请求构造工具，向`http://localhost:8080/servletdemo/DemoServlet`（假设Tomcat在监听8080端口）发一条请求。

如果一切正常的话，就可以看到服务器返回了上面`service()`方法中指定的内容。

# 系列博文

-   [循序渐进写一个Servlet(1) - 介绍相关的接口和类](https://www.boris1993.com/projects/java/Servlet/java-servlet-1-introducing-classes-and-interfaces.html)
-   [循序渐进写一个Servlet(2) - 第一个servlet](https://www.boris1993.com/projects/java/Servlet/java-servlet-2-first-servlet.html)
-   [循序渐进写一个Servlet(3) - 分别处理GET和POST](https://www.boris1993.com/projects/java/Servlet/java-servlet-3-handle-get-and-post.html)
-   [循序渐进写一个Servlet(4) - 会话追踪](https://www.boris1993.com/projects/java/Servlet/java-servlet-4-session-tracking.html)
-   [循序渐进写一个Servlet(5) - Filter](https://www.boris1993.com/projects/java/Servlet/java-servlet-5-filter.html)
