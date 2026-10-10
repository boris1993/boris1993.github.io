---
title: "在Google Cloud Logging的日志输出中增加类名、方法名和行数"
source: https://www.boris1993.com/gcp-logging-show-class-and-method-name.html
date: 2020-08-30
updated: 2026-10-10
tags: [Java, Spring Boot, Google Cloud Logging, Stackdriver]
categories: [小技巧]
---

# 在Google Cloud Logging的日志输出中增加类名、方法名和行数

由于我司目前的项目都运行在Google Cloud Platform(以下简称GCP)上，那么自然而然的，我们选择了使用GCP的Logging来查看日志。在使用过程中，我们发现了一个问题，那就是我们无法直观的看到日志是从什么地方打印出来的，经常需要通过日志内容，在代码里面通过全文搜索来定位。这样就产生了一个需求：可不可以把这条日志所在的类、方法，和行数一起打印在日志中？

## 日志是怎么生成的

既然要实现自动化地修改日志的内容，那么我们首先得要知道，日志的内容是怎么产生的。根据我们的`logback-spring.xml`中的配置，我们可以定位到，日志是从`StackdriverJsonLayout`这个类中生成出来的。

打开这个文件，阅读源码，发现有一个名为`toJsonMap`的方法很像我们这次的目标，为了大家省事，我从GitHub上把这段源码拿到了这里。

```java
/**
 * Convert a logging event into a Map.
 * @param event the logging event
 * @return the map which should get rendered as JSON
 */
@Override
protected Map<String, Object> toJsonMap(ILoggingEvent event) {

    Map<String, Object> map = new LinkedHashMap<>();

    if (this.includeMDC) {
        event.getMDCPropertyMap().forEach((key, value) -> {
            if (!FILTERED_MDC_FIELDS.contains(key)) {
                map.put(key, value);
            }
        });
    }
    if (this.includeTimestamp) {
        map.put(StackdriverTraceConstants.TIMESTAMP_SECONDS_ATTRIBUTE,
                TimeUnit.MILLISECONDS.toSeconds(event.getTimeStamp()));
        map.put(StackdriverTraceConstants.TIMESTAMP_NANOS_ATTRIBUTE,
                TimeUnit.MILLISECONDS.toNanos(event.getTimeStamp() % 1_000));
    }

    add(StackdriverTraceConstants.SEVERITY_ATTRIBUTE, this.includeLevel,
            String.valueOf(event.getLevel()), map);
    add(JsonLayout.THREAD_ATTR_NAME, this.includeThreadName, event.getThreadName(), map);
    add(JsonLayout.LOGGER_ATTR_NAME, this.includeLoggerName, event.getLoggerName(), map);

    // 注意看这个if块，我们的日志语句就是在这里被处理的
    if (this.includeFormattedMessage) {
        String message = event.getFormattedMessage();
        if (this.includeExceptionInMessage) {
            IThrowableProxy throwableProxy = event.getThrowableProxy();
            if (throwableProxy != null) {
                String stackTrace = getThrowableProxyConverter().convert(event);
                if (stackTrace != null && !stackTrace.equals("")) {
                    message += "\n" + stackTrace;
                }
            }
        }
        map.put(JsonLayout.FORMATTED_MESSAGE_ATTR_NAME, message);
    }
    add(JsonLayout.MESSAGE_ATTR_NAME, this.includeMessage, event.getMessage(), map);
    add(JsonLayout.CONTEXT_ATTR_NAME, this.includeContextName, event.getLoggerContextVO().getName(), map);
    addThrowableInfo(JsonLayout.EXCEPTION_ATTR_NAME, this.includeException, event, map);
    addTraceId(event, map);
    add(StackdriverTraceConstants.SPAN_ID_ATTRIBUTE, this.includeSpanId,
            event.getMDCPropertyMap().get(StackdriverTraceConstants.MDC_FIELD_SPAN_ID), map);
    if (this.serviceContext != null) {
        map.put(StackdriverTraceConstants.SERVICE_CONTEXT_ATTRIBUTE, this.serviceContext);
    }
    if (this.customJson != null && !this.customJson.isEmpty()) {
        for (Map.Entry<String, Object> entry : this.customJson.entrySet()) {
            map.putIfAbsent(entry.getKey(), entry.getValue());
        }
    }
    addCustomDataToJsonMap(map, event);
    return map;
}
```

看起来篇幅好像很大，但是仔细一看，不难明白，它就是生成了一个`Map`，而这个`Map`里面放的，就是最后我们在GCP Logging里面将要看到的日志。好了，目标找到了，那就准备动手吧。

## 修改日志内容

要修改的方法找到了，但是我们仍不知道应该怎么取得类名、方法和行号。经过又一顿网上冲浪，加上在源码里面一刨一刨的翻，和一点小小的灵感，最后得知这些信息全部可以通过`ILoggingEvent#getCallerData()`方法取得。请原谅因为时间过去的有点久，我已经找不到当时给我指路的文章，所以也就不能放在下面的参考中了。

那么我们就开始着手实现这个需求吧。首先在项目里面新建一个类，因为我们的目的是重写`toJsonMap`这个方法，所以要让它继承`StackdriverJsonLayout`。

```java
public class GcpLoggingLayout extends StackdriverJsonLayout {
    @Override
    public Map<String, Object> toJsonMap(ILoggingEvent event) {
        // 先调用原本的方法，把日志的信息全部生成好
        Map<String, Object> map = super.toJsonMap(event);

        StackTraceElement[] callerData = event.getCallerData();

        // 如果没有callerData，那后面也没法操作了
        if (callerData == null || callerData.length == 0) {
            return map;
        }

        // 第一个元素就是这个日志所在位置的信息
        StackTraceElement stackTraceElement = callerData[0];

        // 我只需要类名，不需要前面的包名
        // 而 getClassName() 会返回类的全限定名
        // 所以我把它拆了，取最后一个元素，也就是类名
        String[] classNameParts = stackTraceElement.getClassName().split("\\.");
        String callerClass = classNameParts[classNameParts.size - 1];

        String methodName = stackTraceElement.getMethodName();
        String lineNumber = stackTraceElement.getLineNumber().toString();

        // 这个是原本的日志信息
        String originalMessage = map.get(JsonLayout.FORMATTED_MESSAGE_ATTR_NAME);

        // 把类名、方法名、行数附加在原本的信息前面
        String enhancedMessage = "class=" + callerClass + " "
                                + "method=" + methodName + " "
                                + "line=" + lineNumber + " "
                                + originalMessage;

        // 把改好的日志放回去
        map.put(JsonLayout.FORMATTED_MESSAGE_ATTR_NAME, enhancedMessage);

        return map;
    }
}
```

这样我们的日志里就带上了类名、方法名，和行号了。但是到了这里并没有结束，因为上面的代码有一个问题：框架打出来的日志也带上了这些信息。虽然不影响使用，但是它不好看，我不想要。所以我决定继续优化一下。

## 继续优化

稍加思索，我提出了两个新的需求：

-   要根据包名过滤，只给指定的包里面的类加上这些信息。这样，我就可以排除掉所有无关的包。
-   还要根据类名过滤，遇到被指定的类，我就不加上这些信息。因为对于启动类(就是放`main`方法那个)，和一些用于健康检查的类，它们打出的日志也没必要加上这些信息。

但是很明显，原厂的功能并不足以实现这两个需求。我为了这两个需求，新增了两个参数：`appPackage`用来指定应用的包名，`excludedClassNames`用来指定要被排除的类。

增加了参数，那必然也要修改代码。

```java
public class GcpLoggingLayout extends StackdriverJsonLayout {
    private String appPackage;
    private List<String> excludedClassNames;

    public void setAppPackage(String appPackage) {
        this.appPackage = appPackage;
    }

    public String getAppPackage() {
        return this.appPackage;
    }

    // 这里需要注意
    // 我为了 logback-spring.xml 写起来方便，所以在XML中这个属性是一个逗号分隔的字符串
    // 但是在代码里面，我为了用 contains 方法，所以用了一个List来存放
    public void setExcludedClassNames(String excludedClassNames) {
        this.excludedClassNames =
            Arrays.stream(excludedClassNames.split(","))
                .map(String::trim)
                .collect(Collectors.toList());
    }

    // getter就没啥讲究了
    public List<String> getExcludedClassNames() {
        return this.excludedClassNames;
    }

    @Override
    public Map<String, Object> toJsonMap(ILoggingEvent event) {
        // 先调用原本的方法，把日志的信息全部生成好
        Map<String, Object> map = super。toJsonMap(event);

        StackTraceElement[] callerData = event.getCallerData();

        // 如果没有callerData，那后面也没法操作了
        if (callerData == null || callerData.length == 0) {
            return map;
        }

        // 第一个元素就是这个日志所在位置的信息
        StackTraceElement stackTraceElement = callerData[0];

        // 为了通过包名匹配，所以先拿出来全限定名
        String callerClassFullName = stackTraceElement.getClassName();

        // 如果不是应用所在的包打出来的日志，那么不处理
        if (!callerClassFullName.contains(appPackage)) {
            return map;
        }

        String[] classNameParts = stackTraceElement.getClassName().split("\\.");
        String callerClass = classNameParts[classNameParts.size - 1];

        // 如果这个类是被排除的，那就不处理
        if (excludedClassNames.contains(callerClass)) {
            return map;
        }

        String methodName = stackTraceElement.getMethodName();
        String lineNumber = String.valueOf(stackTraceElement.getLineNumber());

        // 这个是原本的日志信息
        String originalMessage = map.get(JsonLayout.FORMATTED_MESSAGE_ATTR_NAME).toString();

        // 把类名、方法名、行数附加在原本的信息前面
        String enhancedMessage = "class=" + callerClass + " "
                                + "method=" + methodName + " "
                                + "line=" + lineNumber + " "
                                + originalMessage;

        // 把改好的日志放回去
        map.put(JsonLayout.FORMATTED_MESSAGE_ATTR_NAME, enhancedMessage);

        return map;
    }
}
```

除了修改代码，日志配置文件`logback-spring.xml`也要针对我们新增的两个属性作出修改。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
  <include resource="org/springframework/cloud/gcp/autoconfigure/logging/logback-appender.xml"/>
  <include resource="org/springframework/cloud/gcp/logging/logback-json-appender.xml"/>
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- CONSOLE输出部分略 -->

  <appender name="CONSOLE_JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      <!-- 指向我们上面写的GcpLoggingLayout -->
      <layout class="com.boris1993.myapplication.config.GcpLoggingLayout">
        <!-- 应用的包名 -->
        <appPackage>com.boris1993.myapplication</appPackage>
        <!-- 要排除的类名 -->
        <!-- 逗号后面的空格可有可无，为了好看我选择加上，反正最后会被trim掉 -->
        <excludedClassNames>APP, HealthCheck</excludedClassNames>
        <!-- 其他GCP Logging配置略 -->
      </layout>
    </encoder>
  </appender>

  <!-- springProfile配置略 -->

</configuration>
```

然后就可以部署到GCP上面看效果了，或者在本机使用对应的profile启动，然后检查控制台里面打印出来的JSON。

\[^1\]: Stackdriver Logging  
\[^2\]: StackdriverJsonLayout.java - GitHub
