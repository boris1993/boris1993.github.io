---
title: "Spring Boot 自动配置的原理"
source: https://www.boris1993.com/java-spring-autoconfiguration.html
date: 2020-02-20
updated: 2026-10-10
tags: [Java, Spring Boot]
categories: [学知识]
---

# Spring Boot 自动配置的原理

在使用Spring Boot时，最使我们收益的一个功能就是它的自动配置。但是，用了这么久的自动配置功能，有没有想过它是怎么实现的？本文将从源码入手，一步一步搞明白Spring Boot自动配置的原理。

我这里就用一个简单的Eureka server的项目来举例。实际上只要是个Spring Boot项目就可以，我只是懒得再创建一个新的项目了。

它的启动代码我们都很熟悉，是这样子的：

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class, args);
    }

}
```

东西很简单，一共就这么几行。我们看看`@SpringBootApplication`里面有什么？

## @SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // 因为里面的属性与本文关系不大，所以就略掉了
}
```

这里我们可以看到一个`@EnableAutoConfiguration`注解，顾名思义，这个注解是用来开启自动配置的。我们继续深入进去看看。

## @EnableAutoConfiguration注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    /**
     * Exclude specific auto-configuration classes such that they will never be applied.
     * @return the classes to exclude
     */
    Class<?>[] exclude() default {};

    /**
     * Exclude specific auto-configuration class names such that they will never be
     * applied.
     * @return the class names to exclude
     * @since 1.3.0
     */
    String[] excludeName() default {};

}
```

这个注解上面还有一大段的JavaDoc，因为篇幅太长，就不把全文放上来了。翻译过来的大意是这样的：

> 这个注解可以启用Spring应用上下文的自动配置。开启这个配置后，Spring将会尝试猜测你需要的类，并进行配置。Spring通常会根据ClassPath和用户定义的bean来完成自动配置的操作。  
> 举例来说，如果在你的ClassPath中出现了tomcat-embedded.jar，那么你很可能会需要TomcatServletWebServerFactory这个类，除非你自己定义了ServletWebServerFactory。

看完JavaDoc，我们大概知道了Spring是怎么判断哪些类需要自动配置。那么我们继续深入源码。

这里有两个注解比较重要：`@Import(AutoConfigurationImportSelector.class)`和`@AutoConfigurationPackage`。

## AutoConfigurationImportSelector类

Spring Boot应用启动过程中使用`ConfigurationClassParser`分析配置类时，如果发现注解中存在`@Import(ImportSelector)`的情况，就会创建一个相应的`ImportSelector`对象， 并调用其方法`public String[] selectImports(AnnotationMetadata annotationMetadata)`。所以我们就从`selectImports`方法开始看起。

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    // 从META-INF/spring-autoconfigure-metadata.properties中加载AutoConfigurationMetaData
    AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);

    // 获取自动配置项
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

上面方法通过`getAutoConfigurationEntry`获取到了需要自动配置的项，那么它是怎么知道哪些东西需要自动配置的呢？

```java
/**
 * Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
 * of the importing {@link Configuration @Configuration} class.
 * @param autoConfigurationMetadata the auto-configuration metadata
 * @param annotationMetadata the annotation metadata of the configuration class
 * @return the auto-configurations that should be imported
 */
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }

    // 获取到注解里面的属性
    AnnotationAttributes attributes = getAttributes(annotationMetadata);

    // 获取到需要自动配置的类，并去重
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);

    // 取得要排除在自动配置之外的类的列表，并将其排除
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);

    // 应用过滤器AutoConfigurationImportFilter
    configurations = filter(configurations, autoConfigurationMetadata);

    // 广播自动配置事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

上面我们看到，Spring Boot通过`getCandidateConfigurations`方法找到了需要自动配置的类，那么它又是怎么工作的呢？

```java
/**
 * Return the auto-configuration class names that should be considered. By default
 * this method will load candidates using {@link SpringFactoriesLoader} with
 * {@link #getSpringFactoriesLoaderFactoryClass()}.
 * @param metadata the source metadata
 * @param attributes the {@link #getAttributes(AnnotationMetadata) annotation
 * attributes}
 * @return a list of candidate configurations
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 扫描ClassPath中所有的META-INF/spring.factories，并从中获取所有自动配置的类名
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
            getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}

/**
 * Return the class used by {@link SpringFactoriesLoader} to load configuration
 * candidates.
 * @return the factory class
 */
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

protected ClassLoader getBeanClassLoader() {
    return this.beanClassLoader;
}
```

上面`getCandidateConfigurations`方法又调用了`loadFactoryNames`方法来获取`EnableAutoConfiguration`注解相关的工厂类。

```java
/**
 * The location to look for factories.
 * <p>Can be present in multiple JAR files.
 */
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

/**
 * Load the fully qualified class names of factory implementations of the
 * given type from {@value #FACTORIES_RESOURCE_LOCATION}, using the given
 * class loader.
 * @param factoryClass the interface or abstract class representing the factory
 * @param classLoader the ClassLoader to use for loading resources; can be
 * {@code null} to use the default
 * @throws IllegalArgumentException if an error occurs while loading factory names
 * @see #loadFactories
 */
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();

    // 根据上文我们知道，factoryClass传入的是EnableAutoConfiguration.class
    // 所以这里就是从spring.factories中寻找org.springframework.boot.autoconfigure.EnableAutoConfiguration对应的自动配置类
    return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

/**
 * 从ClassPath中寻找所有spring.factories文件，并将其包装成一个Properties对象
 * 然后把Properties对象里面的各个条目包装到一个Map<String, List<String>>对象中
 */
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryClassName = ((String) entry.getKey()).trim();
                for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryClassName, factoryName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

## 举例说明

这里我们用`RedisAutoConfiguration`类来说明一个具体的自动配置类是如何工作的。

```java
/**
 * {@link EnableAutoConfiguration Auto-configuration} for Spring Data's Redis support.
 *
 * @author Dave Syer
 * @author Andy Wilkinson
 * @author Christian Dupuis
 * @author Christoph Strobl
 * @author Phillip Webb
 * @author Eddú Meléndez
 * @author Stephane Nicoll
 * @author Marco Aust
 * @author Mark Paluch
 * @since 1.0.0
 */
@Configuration // 说明这是一个配置类
@ConditionalOnClass(RedisOperations.class) // 当RedisOperations类存在时这个配置类才会生效
@EnableConfigurationProperties(RedisProperties.class) // 将配置文件的字段与RedisProperties类绑定
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class }) // 导入两个Redis连接池的配置
public class RedisAutoConfiguration {
    // 这里是初始化redisTemplate和stringRedisTemplate的代码，因为与本文无关，所以略掉了
}
```

根据上文我们了解到的内容，我们可以推测出它是这样子被初始化的：

-   首先Spring在`spring.factories`中，根据`EnableAutoConfiguration`发现了`org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration`
-   然后Spring会检查`RedisOperations`类是否存在于ClassPath中
-   如果存在，则会从配置文件的`spring.redis`字段中取值，并初始化`RedisProperties`
-   然后根据`LettuceConnectionConfiguration`和`JedisConnectionConfiguration`中的条件，选择使用哪个连接池，并将其初始化

## 总结

通过上文的分析，我们知道了Spring在启动时，会加载一系列的配置类，并会根据配置类中指定的条件，来决定是否对其实施自动配置。

`xxAutoConfigurartion`这样的类是自动配置类，用于向容器中添加组件。

`xxProperties`这样的类是属性类，用于封装配置文件中的属性，并规定了Spring应该从配置文件的哪个字段取值用于初始化。
