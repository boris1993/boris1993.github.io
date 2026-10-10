---
title: "循序渐进写一个Servlet(4) - 会话追踪"
source: https://www.boris1993.com/java-servlet-4-session-tracking.html
date: 2019-03-12
updated: 2026-10-10
tags: [Java, Servlet]
categories: [学知识]
---

# 循序渐进写一个Servlet(4) - 会话追踪

Servlet（Server Applet），全称Java Servlet，是用Java编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态Web内容。本系列将一步步地写出一个Servlet程序。

这篇博文将演示如何使用`cookie`和`session`进行会话追踪。

HTTP协议是一个无状态的协议，也就是说，在服务器眼中，每一个HTTP请求都是一个全新的请求，每个请求之间没有关联。所以我们需要一个可以管理请求中携带的用户信息的方法。而会话追踪就是一个可以管理用户信息的方法。

会话追踪可以通过下列几个方式实现：

1.  Cookie
2.  表单隐藏域
3.  URL改写
4.  HttpSession

本文将主要演示`Cookie`和`HttpSession`的用法。

# `Cookie`

## 什么是cookie

`Cookie`是一串可以持久化于各个请求之间的信息片段。每个cookie都有一个名字，并有一个值，同时可以包含备注、路径、域名、过期时间、版本等附加信息。

Cookie有两种：

1.  非持久cookie，这种cookie只在会话中存留，并且不具有过期时间属性，一旦用户关闭浏览器(或者标签页)，也就是使这个会话失效，这个cookie就会丢失。
2.  持久化cookie，这种cookie可以被用于多个会话中，而且只会在到达过期时间，或者用户主动使该cookie失效后，才会被删除。

可以使用`HttpServletResponse#addCookie(Cookie)`方法在HTTP响应中携带cookie。

## 保存cookie

首先修改前文中的`doPost()`方法，将请求中的参数取出来，并存入cookie。

```java
@Override
public void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
    // 设定返回内容的MIME类型
    response.setContentType("text/html");

    // 设定内容以UTF-8编码
    response.setCharacterEncoding("utf-8");

    // 取出所有参数，得到一个Map
    Map parameterMap = request.getParameterMap();

    try (PrintWriter writer = response.getWriter()) {
        // 开始输出HTML文本
        writer.print("<html lang=\"en\">");
        writer.print("<body>");
        writer.print("<b>Response from DemoServlet</b>");
        writer.print("<br>");
        writer.print("<b>Handled by <code>doPost()</code></b>");
        writer.print("<br>");

        // 输出当前session的ID
        writer.print("<b>Session ID: " + request.getSession().getId() + "</b>");
        writer.print("<br>");

        // 遍历parameterMap
        parameterMap.forEach((k, v) -> {
            // 将参数以 key = value 形式输出
            writer.print(k + " = " + ((String[]) v)[0] + "<br>");

            // 将参数的key作为cookie的name，参数的value作为cookie的value
            response.addCookie(new Cookie(String.valueOf(k), ((String[]) v)[0]));
        });

        writer.print("</body>");
        writer.print("</html>");
    }
}
```

然后发送一个`POST`请求，在返回中可以看到请求中的参数已经被放到cookie中，并返回到了客户端。

![POST request with cookie](https://blog-static.boris1993.com/java-servlet-4-session-tracking/post-request-with-cookie.png)

## 使用cookie

一旦cookie被保存到了客户端，那么在下次访问这个cookie所对应的地址时，客户端就会自动将相关的cookie带入请求一并发送到服务端。所以客户端不需要对cookie主动做任何操作。

修改前文中的`doGet()`方法，使其可以取出cookie的值，并输出到页面上。

```java
@Override
public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
    // 设定返回内容的MIME类型
    response.setContentType("text/html");

    // 设定内容以UTF-8编码
    response.setCharacterEncoding("utf-8");

    // 使用Optional类简化null判断
    Optional<String> optionalQueryString = Optional.ofNullable(request.getQueryString());

    // 取出每个参数
    String[] queryStrings = optionalQueryString.isPresent() ? optionalQueryString.get().split("&") : new String[]{};

    try (PrintWriter writer = response.getWriter()) {
        // 开始输出HTML文本
        writer.print("<html lang=\"en\">");
        writer.print("<body>");
        writer.print("<b>Response from DemoServlet</b>");
        writer.print("<br>");
        writer.print("<b>Handled by <code>doGet()</code></b>");
        writer.print("<br>");

        // 输出当前session的ID
        writer.print("<b>Session ID: " + request.getSession().getId() + "</b>");
        writer.print("<br>");

        writer.print("<br>");

        writer.print("<b>Parameters: </b>");
        writer.print("<br>");

        // 遍历每个参数
        for (String query : queryStrings) {
            // 取出参数的key和value
            String[] q = query.split("=");

            writer.print(q[0] + " = " + q[1] + "<br>");
        }

        writer.print("<br>");
        writer.print("<b>Cookies:</b>");
        writer.print("<br>");

        // 从request中取出cookie
        Cookie[] cookies = request.getCookies();

        // 遍历各个cookie
        for (Cookie cookie : cookies) {
            // 输出其name和value
            writer.print(cookie.getName() + " = " + cookie.getValue());
            writer.print("<br>");
        }

        writer.print("</body>");
        writer.print("</html>");
    }
}
```

然后发送一个`GET`请求，在返回中可以看到cookie中的内容已经被输出到页面上。

![GET request with cookie](https://blog-static.boris1993.com/java-servlet-4-session-tracking/get-request-with-cookie.png)

## 删除cookie

将cookie的存活时间设为0，并返回到客户端，即可从客户端中删除这个cookie。

```java
@Override
protected void doDelete(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    Cookie[] cookies = req.getCookies();

    for (Cookie cookie : cookies) {
        // JSESSIONID存放的是当前session的ID
        // 如果删掉这个cookie，那么当前的session也会被丢弃
        if (!"JSESSIONID".equalsIgnoreCase(cookie.getName())) {
            // 设定存活时间为0秒
            cookie.setMaxAge(0);

            // 将修改过的cookie放入响应中返回到客户端
            resp.addCookie(cookie);
        }
    }

    try (PrintWriter writer = resp.getWriter()) {
        // 因为response需要输出到客户端，才可以使新的cookie被送到客户端
        // 但是又懒得输出那么多东西了
        // 所以就输出了一个空字符串
        // 实际上输出内容不影响对cookie的操作
        writer.print("");
    }
}
```

# `HttpSession`

## 什么是session

Session记录着一次会话相关的信息。

当一个请求到达服务器后，服务器会检查请求中是否包含session ID信息，比如在Tomcat中就是检查有无`JSESSIONID`这个cookie，或者URL中有无`JSESSIONID`这个查询字符串。如果找到了对应的session，则服务器会将这个session检索出来使用；请求中没有包含session ID，或者对应的session已经被销毁，则服务器会创建一个新的session并返回其ID。

Session ID通常以cookie的形式返回到客户端，如果客户端禁用了cookie，那么服务端则会使用URL重写技术将session ID写到URL中。

Session中可以键值对的形式保存附加数据，称为attributes。

与cookie不同，session保存于服务器端，而且它能保存的数据也不仅限于字符串。

## 保存attribute

修改`doPost()`方法，编写修改session的代码。修改完成后发送一个带有参数的`POST`请求，以向session中写入一些数据。

```java
@Override
public void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
    // 设定返回内容的MIME类型
    response.setContentType("text/html");

    // 设定内容以UTF-8编码
    response.setCharacterEncoding("utf-8");

    // 取出所有参数，得到一个Map
    Map parameterMap = request.getParameterMap();

    // 获取当前会话的session
    // 如果没有，则会新建一个session并返回其ID
    HttpSession session = request.getSession();

    try (PrintWriter writer = response.getWriter()) {
        // 开始输出HTML文本
        writer.print("<html lang=\"en\">");
        writer.print("<body>");
        writer.print("<b>Response from DemoServlet</b>");
        writer.print("<br>");
        writer.print("<b>Handled by <code>doPost()</code></b>");
        writer.print("<br>");

        // 输出当前session的ID
        writer.print("<b>Session ID: " + request.getSession().getId() + "</b>");
        writer.print("<br>");

        // 遍历parameterMap
        parameterMap.forEach((k, v) -> {
            // 将参数以 key = value 形式输出
            writer.print(k + " = " + ((String[]) v)[0] + "<br>");

            // 将参数的key作为cookie的name，参数的value作为cookie的value
            Cookie cookie = new Cookie(String.valueOf(k), ((String[]) v)[0]);
            response.addCookie(cookie);

            // 将各个参数放到session的attributes中
            session.setAttribute(String.valueOf(k), ((String[]) v)[0]);
        });

        writer.print("</body>");
        writer.print("</html>");
    }
}
```

## 取出attribute

修改`doGet()`方法，使其可以从session中取出attributes并显示在页面上。

```java
@Override
public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
    // 设定返回内容的MIME类型
    response.setContentType("text/html");

    // 设定内容以UTF-8编码
    response.setCharacterEncoding("utf-8");

    // 使用Optional类简化null判断
    Optional<String> optionalQueryString = Optional.ofNullable(request.getQueryString());

    // 获取当前会话的session
    HttpSession session = request.getSession();

    // 取出所有attribute的name
    Enumeration<String> attributeNames = session.getAttributeNames();

    // 取出每个参数
    String[] queryStrings = optionalQueryString.isPresent() ? optionalQueryString.get().split("&") : new String[]{};

    try (PrintWriter writer = response.getWriter()) {
        // 开始输出HTML文本
        writer.print("<html lang=\"en\">");
        writer.print("<body>");
        writer.print("<b>Response from DemoServlet</b>");
        writer.print("<br>");
        writer.print("<b>Handled by <code>doGet()</code></b>");
        writer.print("<br>");

        // 输出当前session的ID
        writer.print("<b>Session ID: " + request.getSession().getId() + "</b>");
        writer.print("<br>");

        writer.print("<br>");

        writer.print("<b>Parameters: </b>");
        writer.print("<br>");

        // 遍历每个参数
        for (String query : queryStrings) {
            // 取出参数的key和value
            String[] q = query.split("=");

            writer.print(q[0] + " = " + q[1] + "<br>");
        }

        writer.print("<br>");
        writer.print("<b>Cookies:</b>");
        writer.print("<br>");

        // 从request中取出cookie
        Cookie[] cookies = request.getCookies();

        // 遍历各个cookie
        for (Cookie cookie : cookies) {
            // 输出其name和value
            writer.print(cookie.getName() + " = " + cookie.getValue());
            writer.print("<br>");
        }

        writer.print("<br>");
        writer.print("<b>Attributes: </b>");
        writer.print("<br>");

        // 遍历attribute的各个name
        while(attributeNames.hasMoreElements()) {
            String key = attributeNames.nextElement();

            // 取出attribute的值
            String value = String.valueOf(session.getAttribute(key));

            writer.print(key + " = " + value);
            writer.print("<br>");
        }

        writer.print("</body>");
        writer.print("</html>");
    }
}
```

然后发送一个`GET`请求，在返回中就可以看到刚才保存在session中的数据：

![GET request with session attribute](https://blog-static.boris1993.com/java-servlet-4-session-tracking/get-request-with-session-attribute.png)

## 删除attribute

此外`HttpSession`类提供了`removeAttribute()`方法用于删除一个attribute。

```java
@Override
protected void doDelete(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    Cookie[] cookies = req.getCookies();

    // 获取当前会话的session
    HttpSession session = req.getSession();

    Enumeration<String> attributeNames = session.getAttributeNames();

    for (Cookie cookie : cookies) {
        // JSESSIONID存放的是当前session的ID
        // 如果删掉这个cookie，那么当前的session也会被丢弃
        if (!"JSESSIONID".equalsIgnoreCase(cookie.getName())) {
            // 设定存活时间为0秒
            cookie.setMaxAge(0);

            // 将修改过的cookie放入响应中返回到客户端
            resp.addCookie(cookie);
        }

        // 遍历attribute names
        while(attributeNames.hasMoreElements()) {
            String key = attributeNames.nextElement();

            // 将其从session中移除
            session.removeAttribute(key);
        }
    }

    try (PrintWriter writer = resp.getWriter()) {
        // 因为response需要输出到客户端，才可以使新的cookie被送到客户端
        // 但是又懒得输出那么多东西了
        // 所以就输出了一个空字符串
        // 实际上输出内容不影响对cookie的操作
        writer.print("");
    }
}
```

# 系列博文

-   [循序渐进写一个Servlet(1) - 介绍相关的接口和类](https://www.boris1993.com/projects/java/Servlet/java-servlet-1-introducing-classes-and-interfaces.html)
-   [循序渐进写一个Servlet(2) - 第一个servlet](https://www.boris1993.com/projects/java/Servlet/java-servlet-2-first-servlet.html)
-   [循序渐进写一个Servlet(3) - 分别处理GET和POST](https://www.boris1993.com/projects/java/Servlet/java-servlet-3-handle-get-and-post.html)
-   [循序渐进写一个Servlet(4) - 会话追踪](https://www.boris1993.com/projects/java/Servlet/java-servlet-4-session-tracking.html)
-   [循序渐进写一个Servlet(5) - Filter](https://www.boris1993.com/projects/java/Servlet/java-servlet-5-filter.html)
