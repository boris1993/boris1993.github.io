---
title: "Spring Boot启动流程分析"
source: https://www.boris1993.com/spring-boot-application-starts-up.html
date: 2024-01-06
updated: 2026-10-10
tags: [Java, Spring Boot]
categories: [学知识]
---

# Spring Boot启动流程分析

想来用了这么久的Spring Boot，但一直没仔细了解它是怎么启动的。那既然想起来了，不如趁热打铁，从它的入口开始，深入看看Spring Boot在启动时都做了些什么。

## 启动入口

入口这部分就没啥说的，跟个Hello world差不多，一个`main`方法执行`SpringApplication#run`来启动整个Spring Boot应用。

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### @SpringBootApplication注解

进到`@SpringBootApplication`注解的源码可以看出，它实际上是`@SpringBootConfiguration`、`@EnableAutoConfiguration`和`@ComponentScan`的组合。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { 
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) 
})
public @interface SpringBootApplication {
    // 略
}
```

#### @SpringBootConfiguration注解

`@SpringBootConfiguration`注解实际上只是`@Configuration`注解的套娃，区别只有两点：

1.  `@SpringBootConfiguration`是Spring Boot提供的注解，而`@Configuration`是Spring提供的注解；
2.  `@SpringBootConfiguration`注解在整个应用中只能出现一次，`@Configuration`注解则可以需要有多少就用多少次。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {
    // 略
}
```

#### @EnableAutoConfiguration注解

`@EnableAutoConfiguration`注解通过引入`AutoConfigurationImportSelector`来开启Spring Boot的自动配置功能。这部分内容我在另一篇博文[Spring Boot 自动配置的原理](https://www.boris1993.com/java-spring-autoconfiguration.html)中有详细的说明，这里就不再重复了。

#### @ComponentScan注解

`@ComponentScan`注解用来配置Spring如何扫描组件。我们可以通过设定`basePackageClasses`或`basePackages`属性来指定从哪些包中扫描，而在不指定的情况下，Spring就会从带有这个注解的类所在的包开始扫描。

因为这个注解会在启动类中被引入，而启动类又在项目最顶层的包中（应该没有谁闲的会去挪启动类的位置吧），所以Spring就会从顶层包开始往下扫描组件。

## SpringApplication类

从启动类对`SpringApplication#run`的调用一路追下去，最后会走到`SpringApplication`类的这个代码块：

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

### SpringApplication的构造方法

可以看到这里先实例化了一个`SpringApplication`对象，那么顺着对应的构造方法一路追下去，最后会看到这样一个构造方法：

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 判断应用程序的类型
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 实例化bootstrap registry initializer
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    // 实例化所有可用的initializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 实例化所有可用的listener
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 找到主类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

#### 判断应用程序的类型

Spring Boot需要判断应用是哪种类型，来决定要不要启动它内嵌的web server，以及启动哪种web server。它会根据这样一个规则来判断当前应用的类型：

-   如果能找到`org.springframework.web.reactive.DispatcherHandler`类，同时找不到`org.springframework.web.servlet.DispatcherServlet`和`org.glassfish.jersey.servlet.ServletContainer`，那么就判定当前应用是一个reactive web应用，并会在将来启动面向reactive的web server；
-   如果`org.springframework.web.servlet.DispatcherServlet`和`org.glassfish.jersey.servlet.ServletContainer`都找不到，说明这个应用不是一个web application，将来也不会启动任何web server；
-   如果以上条件都不符合，那么就判定这个应用是一个servlet web应用，将来会启动面向servlet的web server。

#### 实例化initializer和listener

点进`getSpringFactoriesInstances`方法的实现并顺着追下去，最终可以看到这样一个代码块：

```java
private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);
}
```

再顺着`forDefaultResourceLocation`方法的实现，最终会走到这里：

```java
public static SpringFactoriesLoader forResourceLocation(String resourceLocation, @Nullable ClassLoader classLoader) {
    Assert.hasText(resourceLocation, "'resourceLocation' must not be empty");
    ClassLoader resourceClassLoader = (classLoader != null ? classLoader :
            SpringFactoriesLoader.class.getClassLoader());
    Map<String, SpringFactoriesLoader> loaders = cache.computeIfAbsent(
            resourceClassLoader, key -> new ConcurrentReferenceHashMap<>());
    return loaders.computeIfAbsent(resourceLocation, key ->
            new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)));
}
```

看起来好像云里雾里的，下断点调试一下看出来了，这里就是扫描所有`META-INF`目录下的`spring.factories`文件，并把里面所有的键值对放到一个Map里。最后我们可以得到一个包含了这个Map的`SpringFactoriesLoader`对象。

![](https://blog-static.boris1993.com/how-spring-boot-application-starts-up/load-bootstrap-initializers-1.png)

接着看`load`方法，顺着追下去会走到这个代码块：

```java
public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver,
        @Nullable FailureHandler failureHandler) {

    Assert.notNull(factoryType, "'factoryType' must not be null");
    List<String> implementationNames = loadFactoryNames(factoryType);
    logger.trace(LogMessage.format("Loaded [%s] names: %s", factoryType.getName(), implementationNames));
    List<T> result = new ArrayList<>(implementationNames.size());
    FailureHandler failureHandlerToUse = (failureHandler != null) ? failureHandler : THROWING_FAILURE_HANDLER;
    for (String implementationName : implementationNames) {
        T factory = instantiateFactory(implementationName, factoryType, argumentResolver, failureHandlerToUse);
        if (factory != null) {
            result.add(factory);
        }
    }
    AnnotationAwareOrderComparator.sort(result);
    return result;
}
```

这里就是从上面得到的Map中找到`factoryType`传进来的接口对应的实现类，分别调用它们的构造方法将其实例化，然后把得到的对象放在List里面返回，并交给外层的`setInitializers`方法来把这个List放在`SpringApplication`类的成员变量`initializers`中。

实例化listener也是一样的流程。至于`ApplicationListener`则是Spring的事件监听器，利用观察者模式，通过`ApplicationEvent`和`ApplicationListener`接口实现对Spring容器全生命周期的监听，同时也可以监听自定义的事件。

#### 找到主类

顺着`deduceMainApplicationClass`的实现，会注意到这么两个方法：

```java
private Class<?> deduceMainApplicationClass() {
    return StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE)
        .walk(this::findMainClass)
        .orElse(null);
}

private Optional<Class<?>> findMainClass(Stream<StackFrame> stack) {
    return stack.filter((frame) -> Objects.equals(frame.getMethodName(), "main"))
        .findFirst()
        .map(StackWalker.StackFrame::getDeclaringClass);
}
```

看起来挺简单粗暴的，就是遍历栈帧，找执行了`main`方法的那个栈，然后找到这个栈对应的类。

## run方法

经过上面一顿操作，这个`SpringApplication`对象就初始化好了，接下来就会调用它的`run`方法开始启动。

```java
public ConfigurableApplicationContext run(String... args) {
    Startup startup = Startup.create();
    if (this.registerShutdownHook) {
        SpringApplication.shutdownHook.enableShutdownHookAddition();
    }
    // 创建 bootstrap context
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    // 进入无头模式
    configureHeadlessProperty();
    // 启动listener
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        Banner printedBanner = printBanner(environment);
        // 创建application context
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新application context
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        // 收尾
        startup.started();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), startup);
        }
        listeners.started(context, startup.timeTakenToStarted());
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 略
    }
    try {
        if (context.isRunning()) {
            listeners.ready(context, startup.ready());
        }
    }
    catch (Throwable ex) {
        // 略
    }
    return context;
}
```

### 创建bootstrap context

点进`createBootstrapContext`方法，可以看到这样一个代码块：

```java
private DefaultBootstrapContext createBootstrapContext() {
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
    return bootstrapContext;
}
```

在构造`SpringApplication`时准备的`this.bootstrapRegistryInitializers`在这用上了。看代码的话就是分别执行每个initializer的`initailize`方法。但是我这个应用里面没有`BootstrapRegistryInitializer`的实现类，所以也就没法深入进去看它到底干了什么。

不过看了眼Git Blame，发现了这么一段话：

> Refactor `BootstrapRegistry` support following initial prototype work with the Spring Cloud team.

看起来是跟Spring Cloud相关的，那暂且就不关注了。

### 进入无头模式

在Oracle的文档Using Headless Mode in the Java SE Platform中提到

> Headless mode is a system configuration in which the display device, keyboard, or mouse is lacking. Sounds unexpected, but actually you can perform different operations in this mode, even with graphic data.
> 
> Where it is applicable? Let’s say that your application repeatedly generates a certain image, for example, a graphical authorization code that must be changed every time a user logs in to the system. When creating an image, your application needs neither the display nor the keyboard. Let’s assume now that you have a mainframe or dedicated server on your project that has no display device, keyboard, or mouse. The ideal decision is to use this environment’s substantial computing power for the visual as well as the nonvisual features. An image that was generated in the headless mode system then can be passed to the headful system for further rendering.

其实就是，像web服务之类不需要显示界面的应用，就可以让它进入无头模式，让它在没有显示器等输入输出设备时也能启动，还可以提高计算效率。

### 启动listener

点进`getRunListeners`的实现可以看到如下代码块：

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);
    argumentResolver = argumentResolver.and(String[].class, args);
    List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,
            argumentResolver);
    SpringApplicationHook hook = applicationHook.get();
    SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;
    if (hookListener != null) {
        listeners = new ArrayList<>(listeners);
        listeners.add(hookListener);
    }
    return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);
}
```

看到`getSpringFactoriesInstances`有没有感觉很熟悉？对，这一步就是尝试从`spring.factories`里尝试找到`SpringApplicationRunListener`的实现类。默认来说这里只能找到`EventPublishingRunListener`，它是用来发布各种`SpringBootEvent`的。在Spring Boot中，事件是一个很重要的东西，通过事件机制我们可以监听Spring Boot容器中正在发生的事件，也可以监听各种自定义的事件。事件机制也为Bean之间的消息传递提供支持。

除了从`spring.factories`尝试获取listener之外，Spring Boot也会尝试从`SpringApplicationHook`中找到hook进来的listener。看了下代码，似乎我们可以在入口的`main`方法里调用`SpringApplication#withHook`来添加我们需要的hook，但是我的应用里面也没有什么能用的，所以也没法深挖了。

在得到这些listener之后，就会实例化一个`SpringApplicationRunListeners`对象并返回回去，然后在`SpringApplicationRunListeners#starting`方法中调用各个listener的`starting`方法。此时，上面得到的`EventPublishingRunListener`就会广播出去一条`ApplicationStartingEvent`事件。

```java
void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {
    doWithListeners("spring.boot.application.starting", (listener) -> listener.starting(bootstrapContext),
            (step) -> {
                if (mainApplicationClass != null) {
                    step.tag("mainApplicationClass", mainApplicationClass.getName());
                }
            });
}

private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
        Consumer<StartupStep> stepAction) {
    StartupStep step = this.applicationStartup.start(stepName);
    this.listeners.forEach(listenerAction);
    if (stepAction != null) {
        stepAction.accept(step);
    }
    step.end();
}
```

### 准备环境

Spring中的Environment负责两件事：

-   加载配置好的各种property的值
-   后续通过各种方法获取property的值

在Spring Boot里，property的值可以通过YAML文件或properties文件、环境变量，和命令行参数这三种方法配置。此外，Spring Boot会依照一定的优先级来决定采用哪个property值，高优先级的会覆盖低优先级的。常见的几种配置方式会按照如下的优先级排列：

1.  开发者工具`Devtools`全局配置
2.  命令行指定的参数（如`--server.port=8080`）
3.  JNDI参数
4.  Java系统参数（通过`-D`指定的参数）
5.  系统环境变量
6.  对应不同环境的`application-{profile}.yml`配置文件
7.  `application.yml`配置文件
8.  默认参数

点进`prepareEvent`方法的实现，可以看到这个代码块（我稍微重新格式化了一下，看起来更舒服一点）：

```java
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    DefaultBootstrapContext bootstrapContext, 
    ApplicationArguments applicationArguments
) {
    // Create and configure the environment
    // 获得应用的环境（servlet / reactive）
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(bootstrapContext, environment);
    DefaultPropertiesPropertySource.moveToEnd(environment);
    Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
            "Environment prefix cannot be set via properties.");
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
        environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}

private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    ConfigurableEnvironment environment = this.applicationContextFactory.createEnvironment(this.webApplicationType);
    if (environment == null && this.applicationContextFactory != ApplicationContextFactory.DEFAULT) {
        environment = ApplicationContextFactory.DEFAULT.createEnvironment(this.webApplicationType);
    }
    return (environment != null) ? environment : new ApplicationEnvironment();
}
```

首先在`getOrCreateEnvironment`中，Spring Boot会根据应用类型(reactive、servlet或none)来创建对应的环境。比如我这个是一个servlet应用，那么`createEnvironment`方法就会返回一个`ApplicationServletEnvironment`对象，并返回回去。

接下来到`configureEnvironment`里面：

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    if (this.addConversionService) {
        environment.setConversionService(new ApplicationConversionService());
    }
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}

protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    // 如果有default properties，那就把它们加到MutablePropertySources里面
    if (!CollectionUtils.isEmpty(this.defaultProperties)) {
        DefaultPropertiesPropertySource.addOrMerge(this.defaultProperties, sources);
    }
    // addCommandLineProperties默认是true
    // args就是启动时传进来的参数，--server.port=8080之类的
    // 如果有指定参数，那就把这些参数包在SimpleCommandLinePropertySource里面，并添加到MutablePropertySources中
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
}
```

~这个conversion service是干啥用的暂时还没搞明白，先留个坑，专注于主线。~爬了些文大概看明白了，`ConversionService`是用来处理各种类型转换的，比如把字符串转换成`Long`或者日期等。

这里的话就是把各种参数（比如附加在启动命令里面的命令行参数）给填充到`environment`对象里。`MutablePropertySources`就是存放property的载体，在前面调用`createEnvironment`的时候，`ApplicationServletEnvironment`继承的`AbstractEnvironment`类的构造方法就会创建一个新的`MutablePropertySources`实例。

接下来`ConfigurationPropertySources#attach`方法里面，environment的`propertySources`会被封装成一个`ConfigurationPropertySource`并添加到environment中。

环境准备完成后，Spring Boot会发布一个`ApplicationEnvironmentPreparedEvent`事件。顺着`doWithListeners`方法一路追下去，在`SimpleApplicationEventMulticaster#multicastEvent`方法中，查看`getApplicationListeners(event, type)`的返回结果可以看到，这里会调用多个监听器来处理这个事件。

![](https://blog-static.boris1993.com/how-spring-boot-application-starts-up/get-application-listeners.png)

其中`EnvironmentPostProcessorApplicationListener`又会调用多个后处理器实现加载系统环境变量（由`SystemEnvironmentPropertySourceEnvironmentPostProcessor`完成）、在环境中设定启用的profile并把配置文件加载到环境（由`ConfigDataEnvironmentPostProcessor`完成）等操作。

其中`ConfigDataEnvironmentPostProcessor`默认会从`classpath:/`、`classpath:/config/`、`file:./`、`file:./config/`、`file:./config/*/`这几个地方查找配置文件，如果有需要也可以通过`spring.config.location`、`spring.config.additional-location`、`spring.config.import`这三个配置来干预它从哪查找配置文件。

![](https://blog-static.boris1993.com/how-spring-boot-application-starts-up/default-locations-for-searching-config-files.png)

在查找到配置文件，并得到当前启用的profile后，`ConfigDataEnvironment#applyToEnvironment`方法就会将配置文件中的值加载到环境中。

![](https://blog-static.boris1993.com/how-spring-boot-application-starts-up/loading-config-from-properties-with-profile.png)

然后Spring Boot会把这个environment对象与`SpringApplication`绑定起来。但是绑定这部分暂时也没看明白，依旧是专注于主线，以后有时间再说。

### 创建application context

这里会根据应用的类型（reactive或servlet）来创建对应的application context对象。

```java
protected ConfigurableApplicationContext createApplicationContext() {
    return this.applicationContextFactory.create(this.webApplicationType);
}
```

debug进去，走到了`DefaultApplicationContextFactory`的这两个方法中：

```java
@Override
public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
    try {
        return getFromSpringFactories(
            webApplicationType, 
            ApplicationContextFactory::create,
            this::createDefaultApplicationContext);
    }
    catch (Exception ex) {
        throw new IllegalStateException("Unable create a default ApplicationContext instance, "
                + "you may need a custom ApplicationContextFactory", ex);
    }
}

private <T> T getFromSpringFactories(
    WebApplicationType webApplicationType,
    BiFunction<ApplicationContextFactory, WebApplicationType, T> action, 
    Supplier<T> defaultResult
) {
    for (ApplicationContextFactory candidate : SpringFactoriesLoader.loadFactories(ApplicationContextFactory.class,getClass().getClassLoader())) {
        T result = action.apply(candidate, webApplicationType);
        if (result != null) {
            return result;
        }
    }
    return (defaultResult != null) ? defaultResult.get() : null;
}
```

在这个循环里它会分别执行`ReactiveWebServerApplicationContextFactory`和`ServletWebServerApplicationContextFactory`里面的`create`方法，而`create`方法里会判断当前应用的类型，来决定要不要创建对应的application context。

```java
@Override
public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
    return (webApplicationType != WebApplicationType.SERVLET) ? null : createContext();
}

private ConfigurableApplicationContext createContext() {
    if (!AotDetector.useGeneratedArtifacts()) {
        return new AnnotationConfigServletWebServerApplicationContext();
    }
    return new ServletWebServerApplicationContext();
}
```

创建好application context实例后，Spring Boot会开始准备context的内容。

```java
private void prepareContext(
    DefaultBootstrapContext bootstrapContext, 
    ConfigurableApplicationContext context,
    ConfigurableEnvironment environment, 
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments, 
    Banner printedBanner
) {
    // 绑定环境
    context.setEnvironment(environment);
    // 后置处理
    postProcessApplicationContext(context);
    addAotGeneratedInitializerIfNecessary(this.initializers);
    // 执行各个initializer的initialize方法
    applyInitializers(context);
    // 发布事件
    listeners.contextPrepared(context);
    // 发布bootstrap context被关闭的事件
    bootstrapContext.close(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // 注册启动相关的单例bean
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 把main方法中的args封装成单例bean注册到容器
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    // 设置是否允许循环引用，是否允许覆盖注册
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
        autowireCapableBeanFactory.setAllowCircularReferences(this.allowCircularReferences);
        if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
            listableBeanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
    }
    // 处理延迟初始化
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // 是否要保持JVM持续运行
    // 为了应对Java 21引入的虚拟线程产生的问题
    // 因为虚拟线程都是守护线程，而在只有守护线程运行时，JVM就会退出
    // 所以这里会启动一个非守护线程来保持JVM能持续运行下去
    if (this.keepAlive) {
        context.addApplicationListener(new KeepAlive());
    }
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    // 不考虑AOT优化时，加载primarySources和sources
    if (!AotDetector.useGeneratedArtifacts()) {
        // 载入sources
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        // 加载启动类并注入到容器
        load(context, sources.toArray(new Object[0]));
    }
    listeners.contextLoaded(context);
}
```

#### 注册启动相关的单例bean

我对注册启动相关的单例bean很感兴趣，于是逐层点进`registerSingleton`的实现，看到了下列代码：

```java
// DefaultListableBeanFactory
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    super.registerSingleton(beanName, singletonObject);
    updateManualSingletonNames(set -> set.add(beanName), set -> !this.beanDefinitionMap.containsKey(beanName));
    clearByTypeCache();
}

// DefaultSingletonBeanRegistry
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    Assert.notNull(beanName, "Bean name must not be null");
    Assert.notNull(singletonObject, "Singleton object must not be null");
    synchronized (this.singletonObjects) {
        Object oldObject = this.singletonObjects.get(beanName);
        if (oldObject != null) {
            throw new IllegalStateException("Could not register object [" + singletonObject +
                    "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
        }
        addSingleton(beanName, singletonObject);
    }
}

protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```

看下来就是，在注册一个单例bean的时候，会传进来bean的名字和实际的对象。在`registerSingleton`中首先检查这个名字是不是已经被注册过了，没有被注册过的话就会调用`addSingleton`来注册。而所谓注册，就是：

-   在`singletonObjects`这个Map里面增加一个条目，key是bean的名字，value是bean对应的对象；
-   从`singletonFactories`和`earlySingletonObjects`中删掉以这个bean名字为key的条目；
-   在`registeredSingletons`这个Set里面记录这次注册的bean的名字。

上面提到的这三个Map实际上就是Spring的三级缓存。

-   `singletonObjects`是一级缓存，存储的是完整创建好的单例bean对象。在创建一个单例bean时，Spring会先从这里尝试获取这个bean的实例，如果找到则直接返回，否则继续创建这个bean；
-   `earlySingletonObjects`是二级缓存，存储的是尚未完全创建好的“半成品”单例bean对象。在创建单例bean时，如果发现这个bean存在循环依赖，那么Spring会先创建这个bean的“半成品”对象并将其存到这里。当循环依赖的bean创建完成后，Spring再将这里存储的代理对象替换为完整的bean对象；
-   `singletonFactories`是三级缓存，存储的是单例bean的创建工厂。当一个单例bean被创建时，Spring会先将该bean的创建工厂存储到这里，然后再执行工厂的`getObject()`方法生成该bean的实例对象。在该bean被其他bean引用时，Spring会从这里获取该bean的创建工厂来创建出这个bean的实例，并将这个实例存储到`singletonObjects`中。

返回到`registerSingleton`之后，继续执行`updateManualSingletonNames`方法：

```java
private void updateManualSingletonNames(Consumer<Set<String>> action, Predicate<Set<String>> condition) {
    if (hasBeanCreationStarted()) {
        // Cannot modify startup-time collection elements anymore (for stable iteration)
        synchronized (this.beanDefinitionMap) {
            if (condition.test(this.manualSingletonNames)) {
                Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                action.accept(updatedSingletons);
                this.manualSingletonNames = updatedSingletons;
            }
        }
    }
    else {
        // Still in startup registration phase
        if (condition.test(this.manualSingletonNames)) {
            action.accept(this.manualSingletonNames);
        }
    }
}
```

在应用启动的时候，方法会走到`else`这部分，也就是直接向`manualSingletonNames`这个Set添加这次注册的bean的名字。`manualSingletonNames`这个Set存放的就是手动注册的各个bean的名字。

然后执行`clearByTypeCache`方法，把这两个cache清除。

```java
private void clearByTypeCache() {
    this.allBeanNamesByType.clear();
    this.singletonBeanNamesByType.clear();
}
```

#### 加载primarySources和sources

`getAllSources`会把`primarySources`和`sources`放进一个Set里面返回。

```java
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet<>();
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        // 把启动类添加到allSources
        allSources.addAll(this.primarySources);
    }
    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }
    return Collections.unmodifiableSet(allSources);
}
```

然后走到`load`方法，把bean加载到application context：

```java
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}

private BeanDefinitionRegistry getBeanDefinitionRegistry(ApplicationContext context) {
    if (context instanceof BeanDefinitionRegistry registry) {
        return registry;
    }
    if (context instanceof AbstractApplicationContext abstractApplicationContext) {
        return (BeanDefinitionRegistry) abstractApplicationContext.getBeanFactory();
    }
    throw new IllegalStateException("Could not locate BeanDefinitionRegistry");
}

protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {
    return new BeanDefinitionLoader(registry, sources);
}

// BeanDefinitionLoader
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    // 初始化注解形式的Bean定义读取器
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    // 初始化XML形式的Bean定义读取器
    this.xmlReader = new XmlBeanDefinitionReader(registry);
    this.groovyReader = (isGroovyPresent() ? new GroovyBeanDefinitionReader(registry) : null);
    // 初始化classpath扫描器
    this.scanner = new ClassPathBeanDefinitionScanner(registry);
    // 给classpath扫描器添加排除过滤器
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```

首先这里会执行`getBeanDefinitionRegistry`方法。因为这个应用的application context是一个`AnnotationConfigServletWebServerApplicationContext`，而它又层层继承于`BeanDefinitionRegistry`，所以这里返回的就是当前的application context。得到bean defininition registry之后，就会用它来初始化一个`BeanDefinitionLoader`对象。`BeanDefinitionLoader`的构造方法里面会初始化各种reader和scanner，来支持从不同的资源（XML、Java Config等）加载bean definition。

得到`BeanDefinitionLoader`后，Spring Boot会把bean名字的生成器等自定义部件绑定上去。不过默认情况下它们都是`null`。然后，就会执行`BeanDefinitionLoader`的`load`方法。

```java
void load() {
    for (Object source : this.sources) {
        load(source);
    }
}

private void load(Object source) {
    Assert.notNull(source, "Source must not be null");
    if (source instanceof Class<?> clazz) {
        load(clazz);
        return;
    }
    if (source instanceof Resource resource) {
        load(resource);
        return;
    }
    if (source instanceof Package pack) {
        load(pack);
        return;
    }
    if (source instanceof CharSequence sequence) {
        load(sequence);
        return;
    }
    throw new IllegalArgumentException("Invalid source type " + source.getClass());
}

private void load(Class<?> source) {
    if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
        ((GroovyBeanDefinitionReader) this.groovyReader).beans(loader.getBeans());
    }
    if (isEligible(source)) {
        // 将启动类的BeanDefinition注册进beanDefinitionMap
        this.annotatedReader.register(source);
    }
}

private boolean isEligible(Class<?> type) {
    return !(type.isAnonymousClass() || isGroovyClosure(type) || hasNoConstructors(type));
}
```

虽然这里会遍历`sources`，但是实际上`sources`里面只有应用的主启动类，所以最终会走到`load(Class<?> source)`方法中。因为应用中没有Groovy，所以第一个判断会被跳过，然后经过`isEligible`中的三个判断之后，走进`register`方法，并最终进入`AnnotatedBeanDefinition#doRegisterBean`方法中。

```java
// AnnotatedBeanDefinitionReader
public void register(Class<?>... componentClasses) {
    for (Class<?> componentClass : componentClasses) {
        registerBean(componentClass);
    }
}

public void registerBean(Class<?> beanClass) {
    doRegisterBean(beanClass, null, null, null, null);
}

private <T> void doRegisterBean(
    Class<T> beanClass, 
    @Nullable String name,
    @Nullable Class<? extends Annotation>[] qualifiers, 
    @Nullable Supplier<T> supplier,
    @Nullable BeanDefinitionCustomizer[] customizers
) {
    // 将传进来的类封装为AnnotatedGenericBeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setAttribute(ConfigurationClassUtils.CANDIDATE_ATTRIBUTE, Boolean.TRUE);
    abd.setInstanceSupplier(supplier);
    // 获取该类的scope属性
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    // 设定这个bean的scope，singleton、prototype等
    abd.setScope(scopeMetadata.getScopeName());
    // 生成bean的名字
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 根据类上的注解设定bean的属性，即处理@Lazy、@Primary等
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 将这个BeanDefinition注册到IoC容器的BeanDefinitionMap中
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}

static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }
    else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }

    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }

    AnnotationAttributes role = attributesFor(metadata, Role.class);
    if (role != null) {
        abd.setRole(role.getNumber("value").intValue());
    }
    AnnotationAttributes description = attributesFor(metadata, Description.class);
    if (description != null) {
        abd.setDescription(description.getString("value"));
    }
}
```

这里会处理传进来的类，为它创建一个`BeanDefinition`并设置各种属性，然后调用`BeanDefinitionReaderUtils#registerBeanDefinition`把这个bean注册到application context中。因为这一步传进来的只有主启动类，所以只会注册一个由主启动类生成的bean。

```java
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, 
        BeanDefinitionRegistry registry
    ) throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    // 以bean的名字作为key将其注册到BeanDefinitionMap
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    // 并以这个bean的各个别名再注册一遍
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}

// GenericApplicationContext
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {

    this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
}

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    if (beanDefinition instanceof AbstractBeanDefinition abd) {
        try {
            abd.validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }

    // 检查是否已有同名的bean被注册过了
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        if (!isBeanDefinitionOverridable(beanName)) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            // 如果允许覆盖注册，且当前bean的级别高于已注册的bean（比如已注册的是一个应用自己的bean，但是现在正在注册一个框架提供的bean），
            // 那么就继续
            if (logger.isInfoEnabled()) {
                logger.info("Overriding user-defined bean definition for bean '" + beanName +
                        "' with a framework-generated bean definition: replacing [" +
                        existingDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                        "' with a different definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Overriding bean definition for bean '" + beanName +
                        "' with an equivalent definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        // 覆盖beanDefinitionMap中原有的bean
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (isAlias(beanName)) {
            String aliasedName = canonicalName(beanName);
            if (!isBeanDefinitionOverridable(aliasedName)) {
                if (containsBeanDefinition(aliasedName)) {  // alias for existing bean definition
                    throw new BeanDefinitionOverrideException(
                            beanName, beanDefinition, getBeanDefinition(aliasedName));
                }
                else {  // alias pointing to non-existing bean definition
                    throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                            "Cannot register bean definition for bean '" + beanName +
                            "' since there is already an alias for bean '" + aliasedName + "' bound.");
                }
            }
            else {
                removeAlias(beanName);
            }
        }
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                removeManualSingletonName(beanName);
            }
        }
        else {
            // Still in startup registration phase
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            removeManualSingletonName(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
    else if (isConfigurationFrozen()) {
        clearByTypeCache();
    }
}
```

在启动的时候，实际会走到`registerBeanDefinition`方法的`// Still in startup registration phase`这部分。这里会把传进来的`BeanDefinition`注册到`beanDefinitionMap`中然后返回。

### 刷新application context

在创建application context之后，Spring Boot会刷新它。

```java
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        shutdownHook.registerApplicationContext(context);
    }
    refresh(context);
}

protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
}

// ServletWebServerApplicationContext
public final void refresh() throws BeansException, IllegalStateException {
    try {
        super.refresh();
    }
    catch (RuntimeException ex) {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.stop();
            webServer.destroy();
        }
        throw ex;
    }
}

// AbstractApplicationContext
public void refresh() throws BeansException, IllegalStateException {
    this.startupShutdownLock.lock();
    try {
        this.startupShutdownThread = Thread.currentThread();

        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // Prepare this context for refreshing.
        // 注入listener等组件，激活application context
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 得到当前application context的bean factory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 初始化bean factory，包括设定class loader，注入各种组件等
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 设置beanFactory的后置处理
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // Invoke factory processors registered as beans in the context.
            // 调用beanFactory的后置处理
            // 后置处理器的工作时机是在所有BeanDefinition加载完成之后，bean实例化之前
            // 它可以修改BeanDefinition的属性信息
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 注册bean的后处理器
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // 初始化上下文中的消息源
            initMessageSource();

            // 初始化上下文中的事件机制
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 初始化其他特殊的bean，如创建web server
            onRefresh();

            // Check for listener beans and register them.
            // 把listener beans注入到容器
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 把剩余尚未实例化的bean实例化
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 发布application context刷新完成的事件
            finishRefresh();
        }

        catch (RuntimeException | Error ex ) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            contextRefresh.end();
        }
    }
    finally {
        this.startupShutdownThread = null;
        this.startupShutdownLock.unlock();
    }
}
```

这里首先会一路走到`AbstractApplicationContext#refresh`方法，完成初始化bean factory，实例化剩余的bean等操作。

#### 得到当前application context的bean factory

点进`obtainFreshBeanFactory`方法可以看到这样的实现：

```java
// AbstractApplicationContext
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

// GenericApplicationContext
private final AtomicBoolean refreshed = new AtomicBoolean();

@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}

@Override
public final ConfigurableListableBeanFactory getBeanFactory() {
    return this.beanFactory;
}
```

在`refreshBeanFactory`方法中，Spring会利用`AtomicBoolean`的`compareAndSet`方法来保证这个方法绝对只会被执行一次。

#### 调用beanFactory的后置处理

`invokeBeanFactoryPostProcessors`这里会完成IoC容器初始化的三个步骤，分别是

-   Resource定位  
    在前面Spring Boot已经得到了启动类的`BeanDefinition`，那么在这里它会解析启动类的`BeanDefinition`，得到启动类所在的包并将其作为`basePackage`，这就完成了定位的过程。  
    此外Spring Boot的各种starter是通过SPI机制实现的自动装配，而自动装配也是在这个方法中完成的。  
    还有就是，这个方法中也会处理`@EnableXXX`注解中通过`@Import`指定的配置类。
-   BeanDefinition载入  
    在上一步得到了`basePackage`后，Spring Boot会把路径拼接成`classpath*:com/example/demo/**/*.class`这样的形式，然后`PathMatchingResourcePatternResolver`类会把这个路径下所有的class都加载进来，然后遍历判断有没有`@Component`注解。因为`@Controller`、`@Service`、`@Configuration`之类的实际上只是把`@Component`又包了一层，所以不用单独扫描它们。
-   注册BeanDefinition  
    在这一步，`BeanDefinitionRegister`接口的实现类会把解析到的BeanDefinition向IoC容器注册。

这部分实在是太长，而本文又已经够长了，所以我会[单开一篇来细说](https://www.boris1993.com/spring-boot-application-start-up-post-processing.html)。

#### 把剩余尚未实例化的bean实例化

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no BeanFactoryPostProcessor
    // (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    // 冻结bean definition，不再允许新的修改
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    // 开始实例化bean
    beanFactory.preInstantiateSingletons();
}

public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
            StartupStep smartInitialize = getApplicationStartup().start("spring.beans.smart-initialize").tag("beanName", beanName);
            smartSingleton.afterSingletonsInstantiated();
            smartInitialize.end();
        }
    }
}
```

第一个for循环的重点在于`getBean`方法，逐层点进实现，最终会进入`AbstractBeanFactory#doGetBean`方法。

```java
protected <T> T doGetBean(
    String name, 
    @Nullable Class<T> requiredType, 
    @Nullable Object[] args, 
    boolean typeCheckOnly
) throws BeansException {

    String beanName = transformedBeanName(name);
    Object beanInstance;

    // Eagerly check singleton cache for manually registered singletons.
    // 先检查这个bean是不是一个已经注册过的singleton，并尝试获取。
    // 如果它已经注册过，但是尚未实例化的话，这个方法会将其实例化并返回。
    // 方法里面是一个复杂的双检锁单例模式。
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // 检测bean的正确性，如果获取到的是FactoryBean的话，那么还会调用getObject()方法得到最终的bean实例
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        // 检查容器中是否存在这个bean的BeanDefinition，如果在当前的工厂中找不到，那就到父级BeanFactory中去找，
        // 如果还找不到那就继续往更上一级去找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory abf) {
                // 递归调用父级的doGetBean查找
                return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate").tag("beanName", name);
        try {
            if (requiredType != null) {
                beanCreation.tag("beanType", requiredType::toString);
            }
            // 根据Bean的名字得到它的BeanDefinition
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // 获取当前bean的所有依赖bean。
            // 这里会递归调用getBean()，直到取到一个没有任何依赖的bean
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 如果被依赖的bean也依赖它，那就循环依赖了，没法处理，抛异常
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // Create bean instance.
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        // 创建并注册这个singleton bean
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                if (!StringUtils.hasLength(scopeName)) {
                    throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
                }
                Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new ScopeNotActiveException(beanName, scopeName, ex);
                }
            }
        }
        catch (BeansException ex) {
            beanCreation.tag("exception", ex.getClass().toString());
            beanCreation.tag("message", String.valueOf(ex.getMessage()));
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
        finally {
            beanCreation.end();
            if (!isCacheBeanMetadata()) {
                clearMergedBeanDefinition(beanName);
            }
        }
    }

    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

在检查bean是否存在时调用的`getSingleton()`是这样实现的：

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // Quick check for existing instance without full singleton lock
    // 尝试从缓存中取出这个单例的实例
    Object singletonObject = this.singletonObjects.get(beanName);
    // 如果要获取的bean没有被加载，而且没有正在被创建，那么就主动加载这个bean
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 尝试去earlySingletonObjects里面去找
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                // Consistent creation of early reference within full singleton lock
                // 加锁后再次尝试去singletonObjects和earlySingletonObjects里面找
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // 如果还是找不到，那么就到singletonFactories里面去找对象的实例
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

如果`Object sharedInstance = getSingleton(beanName)`这一步得到的是null，那么容器就会调用`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`方法来创建这个bean。传给`singletonFactory`的lambda表达式中调用了`createBean()`方法，它又调用了`doCreateBean()`方法完成实际的bean创建操作。

##### 创建bean

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {

    // Instantiate the bean.
    // instanceWrapper用来持有创建出来的bean对象
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // 如果是单例的话，那么先把缓存中同名的bean清除
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // 创建bean第一步：创建实例
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.markAsPostProcessed();
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // 当这个bean是一个单例，并且允许循环引用，且这个bean正在创建中时，
    // 就提前暴露一个ObjectFactory，来自动解决循环引用
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 创建bean第二步：设置实例的属性。依赖注入就发生在这里。
        // 如果存在依赖其他bean的属性，那么就递归调用，初始化依赖的bean
        populateBean(beanName, mbd, instanceWrapper);
        // 创建bean第三步：调用bean的初始化方法，如 init-method 指定的方法
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException bce && beanName.equals(bce.getBeanName())) {
            throw bce;
        }
        else {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

那么`doCreateBean()`方法是怎么创建bean的实例的呢？我们继续看`createBeanInstance()`方法。

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point.
    // 首先确认这个bean所属的类可以被实例化
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName, mbd);
    }

    // 如果有工厂方法，那么就用工厂方法来初始化bean
    // 比如配置时在XML中指定factory-method属性，或把带有@Bean注解的方法的返回值作为bean
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    // 如果在重新创建一个bean
    if (resolved) {
        if (autowireNecessary) {
            // 需要调用有参数的构造方法完成注入的话，那就通过构造方法完成依赖注入
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 否则调用默认构造方法
            return instantiateBean(beanName, mbd);
        }
    }

    // Candidate constructors for autowiring?
    // 寻找可以用来初始化的构造方法
    // 如果有的话，那么再逐个匹配，并调用合适的构造方法完成初始化
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // Preferred constructors for default construction?
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }

    // No special handling: simply use no-arg constructor.
    // 否则调用默认构造方法完成实例化
    return instantiateBean(beanName, mbd);
}
```

##### 设置被注入属性的值

完成实例化后就会开始填充这个bean的属性了，继续看`populateBean()`的实现。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    if (bw.getWrappedClass().isRecord()) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to a record");
        }
        else {
            // Skip property population phase for records since they are immutable.
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }

    // 取得在BeanDefinition中设定的property值
    // 这些值可以在XML中用 <property> 属性配置
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        // 用by name的方式完成属性注入
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        // 用by type的方式完成属性注入
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }
    if (hasInstantiationAwareBeanPostProcessors()) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            // 完成@Autowire、@Inject等注解的依赖注入
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                return;
            }
            pvs = pvsToUse;
        }
    }

    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
    if (needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        // 注入配置文件中<property>配置的属性
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

上面代码中`PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName)`这一行完成了`@Autowire`等注解的注入，那么继续看它的实现。

```java
// AutowiredAnnotationBeanPostProcessor
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    // 查找@Autowired、@Inject等注解，并解析出依赖注入的元数据
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        // 完成注入
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

我创建了一个`DemoController`，并在其中用属性注入的方式声明需要注入`DemoService`（`@Autowired private DemoService demoService`），然后在`findAutowiringMetadata`这行下个断点调试，就能看到这样的结果：

![](https://blog-static.boris1993.com/how-spring-boot-application-starts-up/find_autowiring_metadata.png)

具体的注入是这样实现的：

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    // 得到要注入的属性
    Field field = (Field) this.member;
    Object value;
    if (this.cached) {
        try {
            value = resolveCachedArgument(beanName, this.cachedFieldValue);
        }
        catch (BeansException ex) {
            // Unexpected target bean mismatch for cached argument -> re-resolve
            this.cached = false;
            logger.debug("Failed to resolve cached argument", ex);
            value = resolveFieldValue(field, bean, beanName);
        }
    }
    else {
        // 通过BeanFactory解决依赖关系
        value = resolveFieldValue(field, bean, beanName);
    }
    // 解决依赖关系并得到被注入对象的实例后，通过反射把值设置进去
    if (value != null) {
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);
    }
}
```

##### 调用bean的初始化方法

`populateBean()`执行完毕后就会调用`initializeBean()`方法来调用bean的初始化方法（XML或`@Bean`注解配置的`initMethod`，或Bean实现的`InitializingBean#afterPropertiesSet()`方法）

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    // 调用一系列的aware接口实现
    invokeAwareMethods(beanName, bean);

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 初始化前调用BeanPostProcessor
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 调用初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null), beanName, ex.getMessage(), ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 初始化后调用BeanPostProcessor
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware beanNameAware) {
            beanNameAware.setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                beanClassLoaderAware.setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware beanFactoryAware) {
            beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

### 启动部分的收尾工作

在完成刷新application context之后，Spring Boot会发布`ApplicationStartedEvent`和`ApplicationReadyEvent`事件，调用各个`Runner`，然后应用正式启动开始运行

## 参考文章

-   spring boot 启动流程分析
-   spring boot 中的spring factories 机制
-   应用启动过程——准备应用上下文
-   应用启动过程——BootstrapContext
-   SpringApplication中文文档
-   Spring Boot应用Main函数入口 Primary Source
-   Spring源码分析之ConversionService
-   Spring ConversionService 类型转换（一）Converter
-   走心Springboot源码解析： 三、prepareEnvironment()环境配置 解析配置文件信息
