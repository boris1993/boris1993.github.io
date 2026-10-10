---
title: "在Spring中通过配置类注入配置文件的值"
source: https://www.boris1993.com/spring-injecting-property-value-with-configuration-bean.html
date: 2020-05-03
updated: 2026-10-10
tags: [Java, Spring Boot, ConfigurationProperties]
categories: [小技巧]
---

# 在Spring中通过配置类注入配置文件的值

我们在开发过程中，为了保证项目的灵活性，经常会选择将一些值放在配置文件中，并在代码中将它注入并使用。将值注入代码最常见的一种方法，则是使用`@Value()`注解搭配SpEL直接注入我们需要的属性。但是鲁迅先生有云：从来如此，便对吗？这里，我想介绍一个我个人认为更好的实践：通过配置类来注入属性的值。

## 旧的做法有什么问题

假设我们现在有这样一个`application.yml`，其中`credentials`部分是我自定义的一个属性：

```yaml
server:
  port: 9999

credentials:
  token: A_VERY_SECRET_TOKEN
```

然后，我们会在用到它的地方，直接通过`@Value`注解把它注入进来，就像这样：

```java
@Component
public class SomeService {

  @Value("${credientials.token}")
  private String token;
}
```

好像没什么问题对吧，直接用表达式把值拿进来，然后该怎么用就怎么用。但是我不知道你们有没有注意过，这种做法其实既不利于后期重构，也不利于为代码生成好的文档。

比如说，这个值在多个类中都有被引用，但某一天，我们觉得这个名字不够直观，我们想改成`contactServiceAppToken`，那么我们就只能先改掉属性的名字，然后在代码里面全文替换，把`credentials.token`批量替换成`credentials.contactServiceAppToken`。我不知道你们是怎么想的，我每次做这种文本批量替换都很慌，生怕一个没看见而改掉了不应该改的东西。

而对于生成文档，我们都知道，在Java代码上面我们可以使用JavaDoc来编写文档，阐明这个类的作用等等。而对于YAML文件，则没有类似的东西，我们只能在属性上面写普通的注释。可是，大篇幅的注释又有可能会影响YAML文件的可读性，更不用说有谁会在看代码的时候专程去看YAML文件？

所以，我会建议团队使用配置类，也就是本文下面要讲的这个东西，来管理和注入这些自定义的属性。

## 来个示例

首先，我们需要创建一个配置类，来给这些属性找一个家。

```java
// 这个注解是重点，说明我们要把配置文件的 credentials 部分映射到这里
@ConfigurationProperties("credentials")
public class Credentials {
  // 属性名与变量名保持一致即可，Spring会自动处理两者的绑定关系
  // 同时，Spring会自动完成不同命名方式的转换，比如 kebab-case 变成 camelCase
  private String token;

  //getters and setters
}
```

接下来，在要使用这些属性的地方，把这个配置类注入，然后直接get属性的值，就可以了。

```java
@Component
public class SomeService {

  private final Credentials credentials;

  public SomeService(Credentials credentials) {
    this.credentials = credentials;
  }

  public void someMethod() {
    final String token = credentials.getToken();

    // ......
  }
}
```

与直接取值的方法比较起来，使用配置类有这么几个优点：

-   如果在重构的时候要改变属性名，那么我们只需要修改配置文件里面的属性名，和配置类里面的属性名。当然要记得使用IDE里面的重构功能改名，这样IDE会自动分析这个属性的引用，并自动改正过来。
-   使用配置类还可以方便我们生成文档。如果直接在配置文件里面写文档，一方面是不一定易读，另一方面，也不是所有人都会想到在配置文件里面还有文档。而使用配置类的话，我们只需要在类上面加上JavaDoc就好了。
-   而且，我们还不需要担心打错字，导致`@Value`注入失败而使得应用起不来。虽然这不是什么大问题，改正就行了，但毕竟还是麻烦。

## 多层属性怎么办

上面只是演示了只有一级子属性的情况，如果下面包含了多层属性，那配置类应该怎么写呢？

假设现在配置文件变成了这样：

```yaml
credentials:
  token:
    contact: TOKEN_FOR_CONTACT_API
    user: TOKEN_FOR_USER_API
  oauth:
    client-id: CLIENT_ID
    client-secret: CLIENT_SECRET

endpoint:
  contact:
    v1: URL_FOR_CONTACT_API_VERSION_1
    v2: URL_FOR_CONTACT_API_VERSION_2
```

对于`credentials`部分，因为里面子属性的名字大致是确定的，我们用一个内部类就可以搞定（其实写在单独的类里面也可以，只是我不喜欢那么做）。

```java
@ConfigurationProperties("credentials")
public class Credentials {
  private Token token;
  private Oauth oauth;

  //getters and setters
  
  public static class Token {
    private String contact;
    private String user;

    //getters and setters
  }

  public static class Oauth {
    private String clientId;
    private String clientSecret;

    //getters and setters
  }
}
```

取值的时候呢，逐层取到就好了：

```java
@Component
public class SomeService {

  private final Credentials credentials;

  public SomeService(Credentials credentials) {
    this.credentials = credentials;
  }

  public void someMethod() {
    final String contactApiToken = credentials.getToken().getContact();

    // ......
  }
}
```

但是对于`endpoint`部分，因为里面的值是某个API各个版本的URL，考虑到API还有可能会有新版本，每加一个版本都要再改配置类有点麻烦，所以我们可以直接用一个Map来存放。

```java
@ConfigurationProperties("endpoint")
public class Endpoint {
  private Map<String, String> contact;

  //getters and setters
}
```

在取值的时候，就还是一样的套路，注入这个配置类，然后从Map中取值就行了。Map的key就是属性名，比如`v1`，值就是属性的值。当然这样做的话，就要处理一下取到null的情况。

```java
@Component
public class SomeService {

  private final Endpoint endpoint;

  public SomeService(Endpoint endpoint) {
    this.endpoint = endpoint;
  }

  public void someMethod() {
    final String contactV1Url = endpoint.getContact().get("v1");

    if (contactV1Url == null) {
      // handle it here
    }

    // ......
  }
}
```

## 给配置文件加上自动提示

其实，`Configuration properties`配置类除了可以方便我们管理属性之外，他还可以搭配`spring-boot-configuration-processor`来实现配置文件的自动提示，当然这也需要IDE的支持。

在`pom.xml`中加入如下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

然后在编写完配置类之后，执行一下build操作，或者`mvn compile`，来让它帮我们生成一个`additional-spring-configuration-metadata.json`文件。有了这个文件之后，IDE就会参照它在配置文件里面给我们提供自动提示。
