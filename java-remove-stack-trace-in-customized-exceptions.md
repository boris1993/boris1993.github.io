---
title: "去掉自定义异常中的堆栈跟踪信息"
source: https://www.boris1993.com/java-remove-stack-trace-in-customized-exceptions.html
date: 2019-12-17
updated: 2026-10-10
tags: [Java, 代码技巧]
categories: [小技巧]
---

# 去掉自定义异常中的堆栈跟踪信息

```java
/**
 * 业务异常基类
 */
public abstract class BaseBizException extends RuntimeException {
    public BaseBizException(String message) {
        super(message);
    }

    /**
     * 覆盖fillInStackTrace()方法，抹掉异常中的堆栈跟踪信息
     */
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;
    }
}
```
