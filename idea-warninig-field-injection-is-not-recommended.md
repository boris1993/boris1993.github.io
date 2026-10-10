---
title: "IDEA警告Field injection is not recommended"
source: https://www.boris1993.com/idea-warninig-field-injection-is-not-recommended.html
date: 2019-11-29
updated: 2026-10-10
tags: [Java, Spring]
categories: [学知识]
---

# IDEA警告Field injection is not recommended

前些天在开发过程中，发现IDEA在一个`@Autowired`注解上打了一个警告，内容是`Field injection is not recommended`。多年面向Spring开发的经验告诉我，使用`@Autowired`注解进行依赖注入，肯定是没有问题的。但是我的代码洁癖不允许我这么不明不白的留一个警告在这里。所以，带着我的洁癖，和我的好奇心，我开始研究起了这个警告。

## 警告信息

这个警告，和警告的处理建议，在IDEA中是这么写的：

![Warning Message](https://blog-static.boris1993.com/idea-warninig-field-injection-is-not-recommended/warning-message.png)

翻译过来是这个意思：

> 不建议直接在字段上进行依赖注入。  
> Spring开发团队建议：在Java Bean中永远使用构造方法进行依赖注入。对于必须的依赖，永远使用断言来确认。

## 修改代码

既然IDE给了警告，那就先着手修改。一开始，代码是这样子的：

```java
public class AClass{
    @Autowired
    private DependencyClass aDependency;
}
```

根据提示，我将代码修改成了这样子：

```java
public class AClass {
    private final DependencyClass aDependency;

    public AClass(DependencyClass aDependency) {
        this.aDependency = aDependency;
    }
}
```

然后警告就消失了，同时运行没有问题，说明这个修改是可行的。

另外，如果你的项目中引入了`Lombok`，那么代码甚至可以精简成这样子：

```java
// 该注解指示Lombok为所有没被初始化过的final的变量创建构造方法
@RequiredArgsConstructor
public class AClass {
    private final DependencyClass aDependency;
}
```

但是，光是改好代码还远远不够，我需要知道，为什么Spring团队会提出这一项要求，以及，直接使用`@Autowired`进行依赖注入有什么问题。

## 依赖注入的类型

经过我的了解，Spring有三种依赖注入的类型。

### 基于field的注入

所谓基于field的注入，就是在变量上使用`@Autowired`注解进行依赖注入。这是我们最熟悉的一种方式，同时，也正是Spring团队所不推荐的方式。它用起来就像这样：

```java
@Autowired
private DependencyClass aDependency;
```

### 基于setter方法的注入

通过`setter()`方法，以及在方法上加入`@Autowired`注解，来完成的依赖注入，就是基于setter方法的注入。它用起来就像这样：

```java
private DependencyClass aDependency;

@Autowired
public void setADependency(DependencyClass aDependency) {
    this.aDependency = aDependency;
}
```

注：在`Spring 4.3`及以后的版本中，setter上面的`@Autowired`注解是可以不写的。

### 基于构造方法的注入

将各个必需的依赖全部放在带有`@Autowired`注解构造方法的参数中，并在构造方法中完成对应变量的初始化，这种方式，就是基于构造方法的注入。它用起来就像这样：

```java
public class AClass {
    // 这里 final 修饰符并不是必须的，但是我喜欢这么做
    // 因为这样不仅可以在代码上防止 aDependency 被修改
    // 在语义上也可以表明 aDependency 是不应该被修改的
    private final DependencyClass aDependency;

    @Autowired
    public AClass(DependencyClass aDependency) {
        this.aDependency = aDependency;
    }
}
```

注：在`Spring 4.3`及以后的版本中，如果这个类只有一个构造方法，那么这个构造方法上面也可以不写`@Autowired`注解。

## 基于field的注入有什么问题

基于field的注入，虽然不是绝对禁止使用，但是它可能会带来一些隐含的问题。比如，在这篇博客中，作者给出了这样的一个代码：

```java
@Autowired
private User user;

private String school;

public UserAccountServiceImpl(){
    this.school = user.getSchool();
}
```

初看起来好像没有什么问题，`User`类会被作为一个依赖被注入到当前类中，同时这个类的`school`属性将在初始化时通过`user.getSchool()`方法来获得值。但是，这个代码在运行时，却会抛出如下的异常：

```text
Exception in thread "main" org.springframework.beans.factory.BeanCreationException: Error creating bean with name '...' defined in file [....class]: Instantiation of bean failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [...]: Constructor threw exception; nested exception is java.lang.NullPointerException
```

即，在执行`UserAccountServiceImpl()`这个构造方法时出现了NPE。

出现这个问题的原因是，Java在初始化一个类时，是按照`静态变量或静态语句块 –> 实例变量或初始化语句块 –> 构造方法 -> @Autowired`的顺序\[^1\]，那么显而易见，在执行这个类的构造方法时，`user`对象尚未被注入，它的值还是`null`，从而产生了NPE。

此外，在代码质量方面，因为基于field的注入用起来实在是太方便了，增加一个依赖只需要声明一个变量，然后给它加上`@Autowired`注解，就可以了。而这份便利，有可能会导致这个类的依赖变得越来越多，功能越来越杂，最终违反了单一功能原则。这虽然不会导致功能异常，但是这将增大后续维护的难度。（话虽然这么说，就算我用了基于构造方法的注入，但是用Lombok简化了构造方法，这么一来，增加一个依赖又变得更方便了，只需要加一行变量声明就行，如果在不注重代码质量的时候，这也会加剧类的膨胀。所以最后还是得靠工具和审查流程，以及开发者的自觉，来保证代码质量……）

还有一点我个人的感受，就是基于field的注解会占据过多的屏幕空间。按照我个人的代码习惯，每个注入之间都要插入一行空行，来把它们分割开来。这意味着，每个注入都将占据3行。如果这个类有过多的依赖，那么很有可能光是依赖注入的部分，就会占据大半个屏幕，这会让我看起来很不舒服。当然，出现这种情况，可能同时也意味着这个类已经过于膨胀，违反单一功能原则了。

## 基于setter的注入和基于构造方法的注入该怎么选择，有什么优点

对于两种注入方式的取舍，Spring开发团队提供了他们的意见\[^3\]：

> Since you can mix constructor-based and setter-based DI, it is a good rule of thumb to use constructors for mandatory dependencies and setter methods or configuration methods for optional dependencies.

简而言之，`对于必需的依赖，使用基于构造方法的注入；对于可选的依赖，使用基于setter的注入`。

同时Spring开发团队也讲明了两种注入方式的优点。对于基于构造方法的注入，Spring团队是这么说的：

> The Spring team generally advocates constructor injection as it enables one to implement application components as immutable objects and to ensure that required dependencies are not null. Furthermore constructor-injected components are always returned to client (calling) code in a fully initialized state. As a side note, a large number of constructor arguments is a bad code smell, implying that the class likely has too many responsibilities and should be refactored to better address proper separation of concerns.  
> Spring团队提倡使用基于构造方法的注入，因为这样一方面可以将依赖注入到一个不可变的变量中(注：`final`修饰的变量)，另一方面也可以保证这些变量的值不会是null。此外，经过构造方法完成依赖注入的组件(注：比如各个`service`)，在被调用时可以保证它们都完全准备好了。与此同时，从代码质量的角度来看，一个巨大的构造方法通常代表着出现了代码异味，这个类可能承担了过多的责任。

而对于基于setter的注入，他们是这么说的：

> Setter injection should primarily only be used for optional dependencies that can be assigned reasonable default values within the class. Otherwise, not-null checks must be performed everywhere the code uses the dependency. One benefit of setter injection is that setter methods make objects of that class amenable to reconfiguration or re-injection later.  
> 基于setter的注入，则只应该被用于注入非必需的依赖，同时在类中应该对这个依赖提供一个合理的默认值。如果使用setter注入必需的依赖，那么将会有过多的null检查充斥在代码中。使用setter注入的一个优点是，这个依赖可以很方便的被改变或者重新注入。

## 写在最后

虽然上面洋洋洒洒写(chao)了那么多，又是分析优劣，又是分析场景的，但是按照我现在仅有的开发经验来看，好像怎么注入区别都不大(除了setter注入，这个我没用过)，要说我为什么一定要用构造方法注入，最大的原因其实就是为了去掉那个警告……

也有人说，都这么写习惯了，又没出啥问题，你把这个警告关了不就行了吗？我的回答是：

![だが断る！](https://blog-static.boris1993.com/idea-warninig-field-injection-is-not-recommended/but-i-resist.png)

\[^1\]: Spring Bean: Is autowired attribute initialised before constructor?  
\[^2\]: Field Dependency Injection Considered Harmful  
\[^3\]: Setter-based dependency injection
