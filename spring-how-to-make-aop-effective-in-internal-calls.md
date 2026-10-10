---
title: "在同一个类的两个方法内部互相调用中，如何使AOP生效"
source: https://www.boris1993.com/spring-how-to-make-aop-effective-in-internal-calls.html
date: 2025-03-10
updated: 2026-10-10
tags: [Spring, AOP]
categories: [学知识]
---

# 在同一个类的两个方法内部互相调用中，如何使AOP生效

熟悉Spring AOP的都知道，如果同一个类中的两个方法在内部互相调用，那么此时AOP是不会生效的，因为Spring AOP是通过代理类来实现的，而类内部的方法调用并不会走到代理对象。那么，有没有办法让内部调用的时候也让AOP生效呢？万能的ChatGPT告诉我，方法是有的，还有好几种。

## 使用`@Autowired`通过代理调用

这个方法的思路是，利用`@Autowired`注解注入自身的代理对象，然后通过代理对象完成方法调用。

```java
@Service
public class DemoService {
    @Autowired
    private DemoService demoService;

    public String helloWorld() {
        demoService.test();
        return "Hello World!";
    }

    public String test() {
        return "Test";
    }
}
```

这个操作我也在本地的一个Spring Boot项目中验证确实是可行的，前提是要在`application.properties`里面配置允许循环引用`spring.main.allow-circular-references=true`。而且这个操作有一个缺点，就是自身代理对象`demoService`必须通过字段注入的方式完成依赖注入，如果用构造方法注入，启动的时候就会报循环依赖错误导致项目无法成功启动（不用想也知道，在构造方法里面依赖自己肯定不行啊）。

我个人并不喜欢这个方法，一个原因是因为我不喜欢循环引用，另一个原因是我不喜欢字段注入，毕竟Spring早就推荐改成构造方法注入了。所以，我们继续看下一个方法。

## 使用`AopContext.currentProxy()`

另一个方法是使用`AopContext#currentProxy()`静态方法获取到当前的代理对象，也就是对象自己，然后再通过这个代理对象进行方法调用。

```java
@Service
public class DemoService {
    public String helloWorld() {
        final DemoService demoService = (DemoService) AopContext.currentProxy();
        demoService.test();
        return "Hello World!";
    }

    public String test() {
        return "Test";
    }
}
```

但是如果就这样运行，你就会得到这样一条错误信息：

```plaintext
java.lang.IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available, and ensure that AopContext.currentProxy() is invoked in the same thread as the AOP invocation context.
```

看来ChatGPT没把话说全啊，好在我的IDEA里面装了通义灵码，把代码上下文和这个异常抛给它之后，它告诉我要通过注解`@EnableAspectJAutoProxy(exposeProxy = true)`配置Spring AOP允许暴露当前代理对象。那么按照它的说法，我给切面配置类加上这个注解：

```java
@Aspect
@Component
@EnableAspectJAutoProxy(exposeProxy = true) // <-- 就这个，它的默认值是false
public class DemoAopAdvice {
    private static final Logger LOGGER = LoggerFactory.getLogger(DemoAopAdvice.class);

    @Pointcut("execution(* com.example.demo.DemoService.*(..))")
    public void test() {
    }

    @Before("test()")
    public void before() {
        LOGGER.info("Before");
    }
}
```

再重启应用，就发现`demoService`里面的内部调用成功触发了AOP切面。

通义灵码还提醒我，在多线程环境中，要确保`AopContext#currentProxy()`必须在与AOP调用相同的线程中调用。此外，根据`currentProxy()`的JavaDoc，调用它的方法也必须经过了AOP调用，否则会抛出`IllegalStateException`异常。点进`currentProxy()`方法的实现，发现它内部是用ThreadLocal来保存代理对象的，同时在这个类中还有一个`setCurrentProxy(Object)`方法来把当前的代理对象保存到ThreadLocal中。下断点调试后发现，`setCurrentProxy(Object)`这个方法会先被执行，然后再走到我们实际调用的方法。这正好解释了为什么要注意在相同的线程中调用`AopContext#currentProxy()`，并且调用它的方法必须是经过了AOP调用的，因为不这样的话ThreadLocal中根本就没东西可拿。
