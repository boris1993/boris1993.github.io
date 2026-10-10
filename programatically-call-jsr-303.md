---
title: "手动编写代码调用JSR-303 Bean Validation"
source: https://www.boris1993.com/programatically-call-jsr-303.html
date: 2020-01-06
updated: 2026-10-10
tags: [Java, 代码技巧]
categories: [小技巧]
---

# 手动编写代码调用JSR-303 Bean Validation

最近做了一个有点不一样的项目，它是将传入接口的业务参数以JSON的形式放在了一个统一的请求体里面，我要将它取出来，再反序列化到一个Bean里面。这样会带来一个问题，就是我不能直接使用`@Valid`注解来让框架自行校验参数的合法性，而需要手动调用`Validator`实现对bean的校验。

在这里我就不去还原从请求体取出业务数据并反序列化这个过程了，因为这个操作对于我们实际要实现的功能没有关系。我将在这里新建一个简单的类，设定好适当的校验规则，然后通过一个简单的示例来演示。

## 示例bean

示例的bean就是一个喜闻乐见的学生信息，使用`javax.validation.constraints`包中的注解来设定校验规则。

```java
@Data
public class Student {
    // 姓名是必填项
    @NotNull(message = "Student name is mandatory")
    private String name;

    // 性别是必填项，仅接受male和female，首字母可以大写也可以小写
    @Pattern(regexp = "(M|male)|(F|female)", message = "Only male or female are accepted")
    @NotNull(message = "Student gender is mandatory")
    private String gender;

    // 成绩不是必填项
    // 成绩必须大于等于0，且小于等于100
    // 因为Max的值是开区间，所以得写101
    @Max(value = 101, message = "Maximum value of score is 100")
    @PositiveOrZero(message = "Score cannot be negative")
    private Integer score;
}
```

## 编写实现

实现的中心思想就是手动获得一个`Validator`实例，然后调用它来对传入的bean进行校验。

```java
@Slf4j
@RestController
public class StudentController {

    @PostMapping("/student")
    public String showStudent(@RequestBody Student student) {
        // 取得一个Validator实例
        Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

        log.info("Validating bean with validator {}", validator.getClass().getCanonicalName());

        // 调用Validator#validate方法对这个bean进行校验
        // 所有的
        // ConstraintViolation的泛型类型要设定为被校验bean的类型
        Set<ConstraintViolation<Student>> errors = validator.validate(student);

        // 这里遍历errors这个set，打印出各个错误的信息
        errors.forEach(error -> {
            log.error("=======================");
            // 对应校验规则里面的message属性
            log.error("Error message: {}", error.getMessage());
            // 校验失败的属性名
            log.error("Property path: {}", error.getPropertyPath());
            // 导致校验失败的值
            log.error("Error value: {}", error.getInvalidValue());
            log.error("=======================");
        });

        if (errors.size() > 0) {
            // 可以取出所有的校验失败信息，拼接起来之后返回给调用方
            final String errMessages = errors.stream()
                    .map(ConstraintViolation::getMessage)
                    .collect(Collectors.joining(", "));

            // 这里为了省事直接抛出了RuntimeException
            // 实际使用时建议新建一个自定义业务异常代表这种情况
            throw new RuntimeException(errMessages);
        }

        return student.toString();
    }
}
```

## 测试一下

我使用一个这样子的数据来测试上面的校验功能：

```json
{
  "name": "Boris",
  "score": 180
}
```

可见，这个数据是无法通过校验的，它没有填写性别，而且分数超过了上限。请求发出去之后，我得到了这样的错误信息：

```json
{
  "timestamp": "2020-01-06T03:03:18.125+0000",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Maximum value of score is 100, Student gender is mandatory",
  "path": "/student"
}
```

同时，控制台里出现了这样的日志：

```plaintext
2020-01-06 11:21:25.971  INFO 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Validating bean with validator org.hibernate.validator.internal.engine.ValidatorImpl
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error message: Student gender is mandatory
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Property path: gender
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error value: null
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error message: Maximum value of score is 100
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Property path: score
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error value: 180
2020-01-06 11:21:26.214 ERROR 68021 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:21:26.252 ERROR 68021 --- [nio-9999-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: Student gender is mandatory, Maximum value of score is 100] with root cause

java.lang.RuntimeException: Student gender is mandatory, Maximum value of score is 100
    at com.example.demo.StudentController.showStudent(StudentController.java:42) ~[classes/:na]
    堆栈信息太多，下面的略掉了
```

看来，校验的代码成功起作用了。

## 简化代码

因为我们现在基本上都是面向Spring编程，所以其实上面那些手动获取`Validator`的代码也是不必要的。我们可以让Spring自动注入一个`Validator`来实现功能。

```java
@Slf4j
@RestController
@RequiredArgsConstructor
public class StudentController {
    // 直接注入一个Validator的实例
    private final Validator validator;

    @PostMapping("/student")
    public String showStudent(@RequestBody Student student) {
        // 我们来看看它到底注入了谁
        log.info("Validating bean with validator {}", validator.getClass().getCanonicalName());

        // 调用Validator#validate方法对这个bean进行校验
        // 所有的
        // ConstraintViolation的泛型类型要设定为被校验bean的类型
        Set<ConstraintViolation<Student>> errors = validator.validate(student);

        // 这里遍历errors这个set，打印出各个错误的信息
        errors.forEach(error -> {
            log.error("=======================");
            // 对应校验规则里面的message属性
            log.error("Error message: {}", error.getMessage());
            // 校验失败的属性名
            log.error("Property path: {}", error.getPropertyPath());
            // 导致校验失败的值
            log.error("Error value: {}", error.getInvalidValue());
            log.error("=======================");
        });

        if (errors.size() > 0) {
            // 可以取出所有的校验失败信息，拼接起来之后返回给调用方
            final String errMessages = errors.stream()
                    .map(ConstraintViolation::getMessage)
                    .collect(Collectors.joining(", "));

            // 这里为了省事直接抛出了RuntimeException
            // 实际使用时建议新建一个自定义业务异常代表这种情况
            throw new RuntimeException(errMessages);
        }

        return student.toString();
    }
}
```

重新启动应用，并用相同的数据测试之后，我们得到了这样的日志：

```plaintext
2020-01-06 11:15:17.957  INFO 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Validating bean with validator org.springframework.validation.beanvalidation.LocalValidatorFactoryBean
2020-01-06 11:15:18.071 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error message: Student gender is mandatory
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Property path: gender
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error value: null
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error message: Maximum value of score is 100
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Property path: score
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : Error value: 180
2020-01-06 11:15:18.072 ERROR 67745 --- [nio-9999-exec-1] com.example.demo.StudentController       : =======================
2020-01-06 11:15:18.089 ERROR 67745 --- [nio-9999-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: Student gender is mandatory, Maximum value of score is 100] with root cause

java.lang.RuntimeException: Student gender is mandatory, Maximum value of score is 100
    at com.example.demo.StudentController.showStudent(StudentController.java:42) ~[classes/:na]
    下面的堆栈信息依旧略掉
```

看来这种方式使用了另一个`Validator`实现，但是没关系，我们依旧能得到正确的结果，并可以使用完全一样的方法来处理错误信息。

\[^1\]: How to Invoke JSR 303 Bean Validation Programmatically  
\[^2\]: How to manually trigger spring validation? - StackOverflow
