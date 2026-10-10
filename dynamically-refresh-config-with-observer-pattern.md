---
title: "利用观察者模式实现动态刷新Bean中的配置"
source: https://www.boris1993.com/dynamically-refresh-config-with-observer-pattern.html
date: 2025-03-21
updated: 2026-10-10
tags: [设计模式, 观察者模式]
categories: [学知识]
---

# 利用观察者模式实现动态刷新Bean中的配置

公司有的项目里面还是采用把API key明文写在`application.properties`或者某个单独的`properties`文件来管理，这既不安全也不符合公司的规定。所以我通过利用观察者模式和Spring的事件机制，将其改为加密存储于配置中心，并且实现了在配置中心更新后，服务中生效的配置也可以立即更新。

## 背景

谈论实现之前，我先说一下我有哪些工具可以使用：

-   SecretKeyService：一个可以保存对称及非对称加密密钥的服务
-   CredentialVault：一个用来存放密钥的服务，密钥一旦存入，就只能再通过API的方式得到其原文
-   ConfigurationCenter：配置中心，不提供任何加密功能，在配置内容更新后可以通过Spring的事件通知到使用了这个配置的服务

接到这个需求后，我下意识地觉得应该用CredentialVault解决问题，但是仔细想想，发现并不能。因为按照公司要求，API key的密码必须定期轮换，而老的密码在轮换之后就会马上失效。尽管我们有两套API key来避免前面的问题，但是更换API key密钥对又需要修改配置文件，并经历code review及上线部署，依旧需要一定的人工操作。我的设想是，只需要在一个地方设定好要生效的API key，接下来所有用到这个API key的服务都能自动更新。这样看来，似乎带有通知功能的配置中心是唯一解。但是配置中心不提供加密功能，所以还需要SecretKeyService提供一个对称加密密钥来把API key的密码加密，然后将它放到配置中心。

记不住这些名字没关系，后面你也看不到它们了。

## 实现

在确定要用的工具之后，就可以开始着手将它们拼装在一起了。因为原本公司的代码需要保密，所以下面的代码更多是展示思路，大概率你不能直接拷出来放到你的项目中用。

### PropertyLoader

因为直接把配置源从文件换成配置中心有一定的风险，保险起见我们决定逐步迁移配置，在迁移期间需要同时支持配置文件和配置中心。所以，就诞生了`PropertyLoader`。

```java
public class PropertyLoader {
    // 负责加解密的服务，它会从SecretKeyService取得密钥并对给定字符串完成加解密
    private final EncryptionDecryptionService encryptionDecryptionService;
    // 配置中心的客户端
    private final ConfigurationCenterClient configurationCenterClient;
    // 配置文件
    private final Properties properties;

    public PropertyLoader(
        final EncryptionDecryptionService encryptionDecryptionService,
        final ConfigurationCenterClient configurationCenterClient,
        final Properties properties,
    ) {
        this.encryptionDecryptionService = encryptionDecryptionService;
        this.configurationCenterClient = configurationCenterClient;
        this.properties = properties;
    }

    /**
     * 从配置中心或配置文件取得一个key的值
     * 
     * @param key Property或配置中心条目的key
     * @return 取到的值，不会为null
     * @throws PropertyNotFoundException 当在配置中心和配置文件中都找不到这个key时抛出，这通常意味着我们把某条配置漏掉了
     */
    public String loadProperty(final String key) {
        // 首先尝试从配置中心取值，如果取不到则返回null
        String value = configurationCenterClient.getValue(key);
        if (value == null) {
            value = properties.getProperty(key);
        }
        if (value == null) {
            throw new PropertyNotFoundException(key);
        }

        return value;
    }

    /**
     * 从配置中心或配置文件取得一个key的值并将其解密
     * 
     * @param key Property或配置中心条目的key
     * @return 取到的值，不会为null
     * @throws PropertyNotFoundException 当在配置中心和配置文件中都找不到这个key时抛出，这通常意味着我们把某条配置漏掉了
     */
    public String loadEncryptedProperty(final String key) {
        final String encryptedValue = configurationCenterClient.getValue(key);
        if (encryptedValue == null) {
            // 如果从配置中心拿不到值，那就从property文件中拿未加密的原文
            final String plainValue = properties.getProperty(key);
            if (plainValue == null) {
                throw new PropertyNotFoundException(key);
            }

            return plainValue;
        }

        return encryptionDecryptionService.decrypt(encryptedValue);
    }

    /**
     * 检查配置文件或配置中心是否有指定的key
     * 
     * @param key 要检查的key
     * @return 当这个key存在时返回true，反之返回false
     */
    public boolean hasProperty(final String key) {
        return configurationCenterClient.getKeys().contains(key) || properties.containsKey(key);
    }
}
```

### 实现观察者模式

上面说到了，我需要在监听到配置中心发出的事件后，更新相关对象中的配置。显然，在每个对象中都实现一个Spring事件监听器是很蠢的，我们应该在一处监听Spring事件，然后将其以某种方式广播到相关的对象。看起来，观察者模式是个不错的选择。

观察者模式包含两个组件：`notifier`和`subject`。`Subject`作为观众，观察着某个事件；`notifier`则负责将事件通知给各个`subject`。

OK，理论有这些就够了。接下来我们把它实现。首先是`subject`。

#### 观察者模式 - subject

首选我们用一个抽象类定义一个观察者要有的行为，需要成为观察者的类将会继承这个抽象类。

```java
public abstract class ConfigRefreshObserverSubject {
    // 观察到配置更新后，从PropertyLoader获取新的值
    private final PropertyLoader propertyLoader;
    // 这个观察者关心的key
    // 因为要计算在这个对象中当前生效的配置的签名，所以要配置这个对象关心哪些key
    // 然后取出这些key对应的value来计算签名
    private final List<String> monitoredKeys;
    // 当前生效的配置的一个签名
    // 可以将其理解为当前配置的一个哈希，实际上是综合当前生效的配置生成的一个UUID
    // 在收到配置刷新事件后，会将新配置的签名与当前签名比较
    // 仅在签名不一致时刷新对象中的配置
    private String configSignature;

    public ConfigRefreshObserverSubject(
        final PropertyLoader propertyLoader,
        final List<String> monitoredKeys
    ) {
        this.propertyLoader = propertyLoader;
        this.monitoredKeys = monitoredKeys;
        // 在初始化时计算当前生效配置的签名
        this.configSignature = calculateConfigurationSignature();
    }

    /**
     * 这个方法留给实际成为subject的类去实现具体它要怎么刷新自己的配置
     */
    public abstract void refreshConfigImpl();

    /**
     * 这个方法留给notifier调用，实现更新配置及刷新配置签名
     */
    public final void refreshConfig() {
        refreshConfigImpl();
        configSignature = calculateConfigurationSignature();
    }

    public PropertyLoader getPropertyLoader() {
        return this.propertyLoader;
    }

    public String getConfigSignature() {
        return this.configSignature;
    }

    protected String calculateConfigurationSignature() {
        final StringBuilder newConfigurationSignatureSeedBuilder = new StringBuilder();
        for (String key : monitoredKeys) {
            if (!propertyLoader.hasProperty(key)) {
                // 如果找不到某个关心的key，那么说明要么初始配置有问题，要么刷新的配置有问题
                // 这时候尽早抛出异常引发开发人员关注
                throw new IllegalArgumentException("Missing property: " + key + " for class: " + this.getClass().getSimpleName());
            }

            newConfigurationSignatureSeedBuilder.append(propertyLoader.loadProperty(key));
        }

        return UUID.nameUUIDFromBytes(newConfigurationSignatureSeedBuilder.toString().getBytes()).toString();
    }
}
```

#### 观察者模式 - notifier

在观察者模式中，`notifier`将作为一个单例存在，各个`subject`会注册到这个`notifier`，并在接收到事件后被`notifier`逐个通知。

```java
public class ConfigRefreshObserverNotifier {
    // 用于存放各个subject的注册表
    // 在某些情况下，同一个subject可能会被重复注册，所以这里我会把subject的类名作为key放在map中
    // 方便在注册时检查是否重复注册
    private static final Map<String, ConfigRefreshObserverSubject> subjects = new HashMap<>();

    private ConfigRefreshObserverNotifier() {
        // notifier作为一个单例，我们不希望它被实例化
        throw new UnsupportedOperationException("ConfigRefreshObserverNotifier shouldn't be instantiated");
    }

    /**
     * 注册subject
     * @param subject 待注册的subject对象
     */
    public static void register(final ConfigRefreshObserverSubject subject) {
        if (!isSubjectRegistered(subject)) {
            subjects.put(subject.getClass().getSimpleName(), subject);
        }
    }

    /**
     * 向各个subject发出配置更新的通知
     */
    public static void notifyObservers() {
        for (final ConfigRefreshObserverSubject subject : subjects.values()) {
            // 计算新配置的签名
            final String newConfigSignature = subject.calculateConfigurationSignature();
            // 仅当签名不同，即配置有变化时，才通知对应的subject更新
            if (!newConfigSignature.equals(subject.getConfigSignature())) {
                subject.refreshConfig();
            }
        }
    }

    private static boolean isSubjectRegistered(final ConfigRefreshObserverSubject subject) {
        final String subjectClassName = subject.getClass().getSimpleName();
        return subjects.containsKey(subjectClassName);
    }
}
```

#### 观察者模式 - 实际的观察者类

有了`subject`，接下来就可以让实际要监控配置更新的类继承`ConfigRefreshObserverSubject`，将它变为一个观察者，并实现更新配置的逻辑。这部分其实很简单，就是给对应的字段重新赋值。

```java
public class SomeServiceClient extends ConfigRefreshObserverSubject {
    private String endpoint;
    private String username;
    private String password;

    public SomeServiceClient(final PropertyLoader propertyLoader) {
        super(
            propertyLoader,
            List.of("some-service-endpoint", "some-service-username", "some-service-password")
        );

        initializeProperties();
        ConfigRefreshObserverNotifier.register(this);
    }

    private void initializeProperties() {
        final PropertyLoader propertyLoader = super.getPropertyLoader();

        this.endpoint = propertyLoader.loadProperty("some-service-endpoint");
        this.username = propertyLoader.loadProperty("some-service-username");
        this.password = propertyLoader.loadEncryptedProperty("some-service-password");
    }

    @Override
    public void refreshConfigImpl() {
        initializeProperties();
    }

    // 实际这个类的业务实现与本文无关，就省略了
    // 说白了无非就是往endpoint发请求，带上username和password来认证
}
```

上面的代码应该很容易理解，在配置刷新前，`SomeServiceClient`就用当前生效的配置去发请求，在配置刷新后，这个观察者就可以马上得知这个事件并从配置中心取得最新的值替换当前生效的配置。这个过程几乎是瞬间完成的，不会对业务产生影响。

### 监听配置刷新事件

上面洋洋洒洒实现了一堆东西，但最重要的一个还没有实现，那就是配置刷新事件的监听器。因为配置中心会通过Spring事件来发布，所以只需要找个地方实现一个`@EventListener`方法就行。

```java
@Configuration
public class BeanFactory {
    @Bean
    public SomeServiceClient someServiceClient(
        final ResourceLoader resourceLoader,
        final EncryptionDecryptionService encryptionDecryptionService 
    ) {
        final Properties properties = ResourcesUtils.loadProperties(
            resourceLoader,
            ResourcesUtils.CLASSPATH_META_INF + "/some-service.properties"
        );

        final PropertyLoader propertyLoader = new PropertyLoader(encryptionDecryptionService, properties);
        
        return new SomeServiceClient(propertyLoader);
    }

    @EventListener
    public void handleConfigRefreshEvent(final ConfigRefreshEvent event) {
        if (event.getProjects().contains("my-config-project")) {
            ConfigRefreshObserverNotifier.notifyObservers();
        }
    }
}
```

以上，就完成了一个利用配置中心的通知机制实现的配置动态更新功能。
