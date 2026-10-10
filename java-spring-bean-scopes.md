---
title: "Spring bean的作用域"
source: https://www.boris1993.com/java-spring-bean-scopes.html
date: 2020-02-22
updated: 2026-10-10
tags: [Java, Spring Boot]
categories: [学知识]
---

# Spring bean的作用域

在Spring中，那些由IoC容器所管理的对象被称之为bean。而一个bean的定义，其实只是一个“蓝图”，指导着Spring如何去创建这样一个bean。而在这个蓝图中，有一个属性叫做“作用域”，它规定了这个bean的可见范围。这里我们看一下Spring的bean都有哪些作用域。

## 支持的作用域

我们先来看一下Spring支持哪些作用域。

| 作用域 | 说明 |
| --: | --- |
| singleton | 在Spring容器中仅存在一个bean的实例，bean以单例形式存在。这是默认的作用域 |
| prototype | 每次从容器中获取bean时，都将生成一个新的实例，即相当于每次都执行`new xxxBean()` |
| request | 在HTTP请求(request)的完整生命周期中，将创建并使用单个实例。该作用域仅适用于`WebApplicatonContext`环境 |
| session | 在HTTP会话(session)的完整生命周期中，将创建并使用单个实例。该作用域仅适用于`WebApplicationContext`环境 |
| globalSession | 在全局的HTTP会话(session)的完整生命周期中，将创建并使用单个实例。该作用域仅适用于`WebApplicationContext`环境，且通常只能用在`Portlet`环境中。 |
| application | 在`ServletContext`的完整生命周期中，将创建并使用单个实例。该作用域仅适用于`WebApplicationContext`环境 |
| websocket | 在`WebSocket`的完整生命周期中，将创建并使用单个实例。该作用域仅适用于`WebApplicationContext`环境 |

## 指定bean的作用域

要指定一个bean的作用域，我们可以通过XML的方式或注解的方式来设定。

使用XML指定配置bean时，可以通过`scope`属性来指定作用域：

```xml
<bean id="someBean" class="com.demo.SomeClass" scope="singleton"/>
```

使用注解方式配置bean时，可以通过`@Scope`注解来指定作用域：

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
public class SomeClass {
    // Class definitions goes here
}
```

此外，如果使用注解方式配置作用域，Spring也提供了一系列常量值来方便我们配置：

```java
// 在ConfigurableBeanFactory类中
String SCOPE_SINGLETON = "singleton";
String SCOPE_PROTOTYPE = "prototype";

// 在WebApplicationContext类中
String SCOPE_REQUEST = "request";
String SCOPE_SESSION = "session";
String SCOPE_APPLICATION = "application";
```

## singleton作用域

`singleton`是Spring容器中的默认作用域。这个作用域下，容器中只创建各管理一个bean实例，实例存在于缓存中，并在后续对该bean的请求中都返回这个实例。

## prototype作用域

与`singleton`正相反，每次对`prototype`作用域的bean的请求，Spring都会生成一个新的实例，即类似我们手动使用`new XxxBean()`方式创建实例。

需要注意的是，Spring不会完整的管理一个`prototype`的bean的生命周期。容器在初始化、配置，并将bean交由请求方(client)之后，就撒手不管了。也就是说，在销毁一个`prototype`的bean时，销毁bean的回调方法是不会被调用的，所以在销毁一个`prototype`的bean时，开发者必须手动释放它所使用的资源，或者可以尝试使用一个自定义的[`bean post-processor`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-extension-bpp)来让Spring做这些事。

对于有状态的bean，应当使用`prototype`作用域；对于无状态的bean，则应当使用`singleton`作用域。

## 向singleton bean注入prototype bean

因为bean的依赖关系在实例化bean时才会被解析，所以通常来说，我们不可以将一个prototype bean注入到一个singleton bean中。

如果我们向一个singleton bean中注入一个prototype bean，因为这个singleton bean只会被实例化一次，使得它的依赖也只会被注入一次，最终导致它依赖的那个singleton bean也只存在一个实例。

## request、session、global session、application和websocket作用域

这几种作用域只能用在`web-aware`的Spring上下文中，比如`XmlWebApplicationContext`。如果用在一般的IoC容器中，比如`ClassPathXmlApplicationContext`中，那么容器会抛出一个`IllegalStateException`。

要使用这几个作用域，你可能需要对你的应用进行一些配置。因为这些内容与本文无关，所以在这里就不详细说明了。感兴趣的话可以看Spring参考手册中的内容。

注：`web-aware`这个词，我也不知道怎么翻译才合适。查阅了一些资料之后，感觉一个`web-aware`的Spring应用就是一个运行在web容器(比如Tomcat)中的应用，因为上面提到的这些作用域也是与web应用相关的。如果有好的理解，请一定在留言区写下来让在下知道。

### request作用域

`request`作用域下的bean，在每次HTTP请求中，都会创建一个新的实例。当请求完成时，对应的bean就会被销毁。对一个实例的任何更改，对其他的所有实例来说都是不可见的。

### session作用域

`session`作用域下的bean，在每个活动的HTTP会话中，都有一个独自的实例，而当会话结束后，对应的bean就会被销毁。对一个实例的任何更改，对其他所有的实例来说都是不可见的。

### globalSession作用域

这个作用域只能用在`portlet`应用中。一个`portlet`站点中可能有多个`portlet`应用，而它们相关的session中都会共享同一个`globalSession`作用域的bean。

注：其实我也不知道`portlet`到底是个啥，就算看过维基百科的Portlet条目也没看明白。

### application作用域

在整个应用范围内，容器为每个web应用程序运行时创建一个实例。这个作用域与`singleton`很类似，但是还是有两个不同点：

-   在不同`ServletContext`中有不同的bean单例对象；singleton作用域的bean是每个`ApplicationContext`的单例对象。而一个应用可能有多个`ApplicationContext`
-   bean作为`ServletContext`属性可见

\[^1\]: Bean Scopes - The IoC container  
\[^2\]: Spring系列四：Bean Scopes作用域  
\[^3\]: Spring学习（十五）Spring Bean 的5种作用域介绍
