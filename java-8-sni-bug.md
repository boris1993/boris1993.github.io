---
title: "Java 8的一个SNI的bug"
source: https://www.boris1993.com/java-8-sni-bug.html
date: 2021-11-24
updated: 2026-10-10
tags: [Java 8, bug, SNI]
categories: [学知识]
---

# Java 8的一个SNI的bug

联动[上一篇博文](https://www.boris1993.com/tools/nginx/resolving-nginx-sni-issue.html)，在解决了OpenResty上那个SNI的问题之后，我们发现有一个Java应用也有类似的问题。而最后发现，这是因为我们当前版本的Java中有一个bug……

简单来说，就是在`Java 1.8u141`之前，`HttpsURLConnection#setDefaultHostnameVerifier()`方法会破坏SNI，而正好我们的代码里有这么一行：

```java
HttpsURLConnection.setDefaultHostnameVerifier((hostname, session) -> true);
```

翻了下框架的代码，发现如果我不设定这个值，框架也调用这个方法来指定它默认的hostname verifier；而如果我传个null，那么它会抛个异常给我。所以，代码层面没有很方便的解决方案。

无奈，最后决定，先临时换了个没开启SNI的域名，暂时解决掉问题，然后升级Java到`1.8u181`，一劳永逸。

* * *

终于，我也有机会理直气壮地喊出“这是Java的bug”了！（笑

\[^1\]: Extended server\_name (SNI Extension) not sent with jdk1.8.0 but send with jdk1.7.0  
\[^2\]: Custom HostnameVerifier disables SNI support on client in Java 8
