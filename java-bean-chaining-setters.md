---
title: "在Java bean中实现setter的链式调用"
source: https://www.boris1993.com/java-bean-chaining-setters.html
date: 2019-09-24
updated: 2026-10-10
tags: [Java, 代码技巧]
categories: [小技巧]
---

# 在Java bean中实现setter的链式调用

在经年累月的开发中，你是不是已经对动辄数行的`.setXXX()`方法厌烦了呢？在这篇博文中，我将介绍一个小技巧，可以稍稍让你的代码变得美观一些。

## 前言

通常来说，一个POJO大致上会是这样的

```java
public class Student {
    private String name;
    private Integer age;
    private Integer score;

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return this.age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Integer getScore() {
        return this.score;
    }

    public void setScore(Integer score) {
        this.score = score;
    }
}
```

然后，在赋值的时候，会是这样的

```java
Student james = new Student();
james.setName("James");
james.setAge(22);
james.setScore(80);
```

上面这三行set方法，本来没干什么事，却要写3行代码六十多个字，是不是看起来很难受，写起来很费劲？

那么，这时候就可以通过链式调用setter来显著减少代码行数。

## Setter的链式调用

首先，要实现链式调用setter，就得稍微修改一下setter方法。放心，改动量是很小的，对业务是没有影响的，操作也是一劳永逸的。

具体做法呢，就是让setter返回这个POJO类型，而不是void，然后将`this`返回，比如这样：

```java
public Student setName(String name) {
    this.name = name;
    return this;
}

public Student setAge(Integer age) {
    this.age = age;
    return this;
}

public Student setScore(Integer score) {
    this.score = score;
    return this;
}
```

然后，在创建对象的时候，就可以这样：

```java
Student student = new Student().setName("james").setAge(22).setScore(80);
```

怎么样？是不是突然变得很短很简洁？写起来是不是很方便？

但是，要一个个改那么多setter，也挺麻烦的，特别是IDE里面还不一定能自动化，有没有更好的办法？

答案是，有！

是啥呢？

Lombok！

## 用Lombok实现setter的链式调用

Lombok是一个自动生成模版代码的工具，像构造方法啊、getter和setter啊，都可以用一个或多个注解来代替，然后在编译期由Lombok替你写这部分的代码。如果你还没用过，那么请务必学习和试用一下，它可以为你省下那些浪费在模版代码上的时间。

在引入了Lombok之后，你就可以把上文的`Student`类重构成这样子了：

```java
@Data
@Accessors(chain = true)
public class Student {
    private String name;
    private Integer age;
    private Integer score;
}
```

是的，你不再需要去写getter和setter了，更不需要一个个的去修改来让它支持链式调用了。只需要两个注解，就可以完成上面的一大片工作。

第一行的`@Data`注解，是一系列注解的集合，用来生成一系列的模版代码。由于这不是本文重点，所以就不再展开说了。

第二行的`@Accessors`注解，就是这里的重点了，它用来设定Lombok应该怎样生成setter和getter代码。而其中的`chain`属性，就是打开链式setter的开关，将它打开之后，就可以生成支持链式调用的setter方法。

然后，你就可以像上文一样，去享受链式调用了。
