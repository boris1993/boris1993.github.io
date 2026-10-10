---
title: "Java自定义注解"
source: https://www.boris1993.com/java-customized-annotations.html
date: 2019-03-15
updated: 2026-10-10
tags: [Java, 注解, Annotations]
categories: [学知识]
---

# Java自定义注解

注解是Java 5引入的一个新特性，它提供了一个用来将信息和元数据与程序元素相关联的能力，其作用如同一个修饰符，本身并不包含任何程序逻辑。

本文将介绍如何创建和使用自定义的注解。

# 元注解

Java自带了四个作用于注解上的注解，即元注解，分别是：

-   `@Documented`，用于注明该注解是否包含于JavaDoc中
-   `@Retention`，用于注明这个注解将保留到什么时候
-   `@Target`，用于注明这个注解将作用于哪些元素上
-   `@Inherit`，用于注明该注解是否会被子类继承

## `@Retention`

`@Retention`元注解定义了这个注解的生命周期，即这个注解将保留到什么时候。注解的生命周期有这三种：

-   `RetentionPolicy.SOURCE`：仅在源码中保留，在编译期就会被丢弃。比如`@Override`和`@SuppressWarnings`就属于这类注解
-   `RetentionPolicy.CLASS`：注解将会被写入到字节码中，但是在运行时会被丢弃。这个是默认的生命周期。
-   `RetentionPolicy.RUNTIME`：该注解将保留至运行时。这意味着在运行时可以通过反射机制读取到注解的信息。

## `@Target`

`@Target`元注解指定了该注解将可用于哪些元素上。可用的参数有如下几种：

-   `ElementType.ANNOTATION_TYPE`，用于描述注解。`@Target(ElementType.ANNOTATION_TYPE)`标注的注解将成为一个元注解。
-   `ElementType.CONSTRUCTOR`，用于描述构造方法
-   `ElementType.FIELD`，用于描述成员变量、对象、属性（包括enum实例）
-   `ElementType.LOCAL_VARIABLE`，用于描述局部变量
-   `ElementType.METHOD`，用于描述方法
-   `ElementType.PACKAGE`，用于描述包
-   `ElementType.PARAMETER`，用于描述参数
-   `ElementType.TYPE`，用于描述类、接口（包括注解）、enum生命声明

Java 8中又新增了两个参数：

-   `ElementType.TYPE_PARAMETER`，可以用在Type的声明前
-   `ElementType.TYPE_USE`，可以用在使用Type的地方

# 编写自定义注解及相关方法

自定义注解的类型为`@interface`，注解中可以包含方法，方法名将作为注解的属性。

注解中的方法不可以有参数，也不可以抛出异常，同时方法只能返回原始类型、`String`、`Class`、`enums`、注解类型，以及上述类型的数组。方法的默认值不可以是`null`。

下面将通过一个示例演示如何编写和使用自定义注解相关的方法。

示例将分别创建两个名为`@JsonSerializable`和`@JsonElement`的注解，以及一个名为`JsonUtils`的工具类。

`@JsonSerializable`标记一个类可以被序列化成JSON，`@JsonElement`标记一个成员变量将会被包含在这个JSON中；`JsonUtils`工具类包含将对象序列化为JSON的方法。

## `@JsonSerializable`

```java
/**
 * 标记一个类可以被序列化成JSON
 *
 * 因为这个注解要在运行时通过反射获取，所以retention为runtime
 *
 * 因为这个注解作用于一个类，所以target为type
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface JsonSerializable {
}
```

## `@JsonElement`

```java
/**
 * 标记一个成员变量将会被包含在这个JSON中
 *
 * 因为这个注解要在运行时通过反射获取，所以retention为runtime
 *
 * 因为这个注解作用于成员变量，所以target为field
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface JsonElement {

    /**
     * 指定该成员变量在JSON中的key值
     */
    public String key() default "";
}
```

## `JsonUtils`

```java
public class JsonUtils {

    /**
     * 将对象序列化为JSON
     *
     * @param object 要序列化的对象，需要有{@link JsonSerializable}注解
     * @return 序列化后的JSON字符串，如果不可序列化则是null
     * @throws InvocationTargetException
     * @throws IllegalAccessException
     * @throws NoSuchMethodException
     */
    public static String serializeToJson(Object object) throws InvocationTargetException, IllegalAccessException, NoSuchMethodException {

        // 检查对象是否可以被序列化
        if (!isSerializable(object)) {
            return null;
        }

        // 取得对象所属的类
        Class clazz = object.getClass();

        // 取得类中的所有方法
        Method[] methods = clazz.getMethods();

        // 取得所有方法的方法名，后面用于搜索
        List<String> methodNames = Arrays.stream(methods).map(Method::getName).collect(Collectors.toList());

        // 取得类中所有成员变量，包括public、protected、private、和默认访问权限的
        Field[] fields = clazz.getDeclaredFields();

        // 创建一个空的HashMap，用于存放要序列化的属性的名字和值
        Map<String, String> elements = new HashMap<>(fields.length);

        // 遍历所有成员变量
        for (Field field : fields) {
            // 如果有JsonElement注解
            if (field.isAnnotationPresent(JsonElement.class)) {

                // 取得变量名
                String fieldName = field.getName();

                // 拼接其对应getter方法名
                // 不直接使用setAccessible()方法是因为我不喜欢这么干，这会破坏封装性
                String getterName = "get" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);

                // 检查这个变量是否有getter方法
                if (methodNames.contains(getterName)) {

                    // 如果有getter方法，则根据方法名取得对应的方法实例
                    Method method = clazz.getMethod(getterName);

                    // 取得JsonElement注解中设定的key值
                    String keyName = field.getAnnotation(JsonElement.class).key();

                    // 如果key的值为空字符串，则使用属性名作为JSON中的key名
                    // 否则取指定的key名
                    // 并调用变量对应的getter方法取得变量的值
                    // 最后放入HashMap中
                    elements.put("".equals(keyName) ? field.getName() : keyName, String.valueOf(method.invoke(object)));
                }
            }
        }

        // 遍历HashMap，构造JSON内容
        String jsonBody = elements.entrySet()
                .stream()
                // 取得每个元素的key名和值，拼接成 \t"key":"value" 的形式
                .map(entry -> "\t\"" + entry.getKey() + "\":\"" + entry.getValue() + "\"")
                // 每行元素间插入分隔符，逗号分隔每行数据，\n实现换行
                .collect(Collectors.joining(",\n"));

        // 最后拼接JSON首尾的大括号
        return "{\n" + jsonBody + "\n}";
    }

    /**
     * 检查对象是否可被序列化成JSON
     * @param object 将被序列化的对象
     * @return 是否可被序列化
     */
    private static boolean isSerializable(Object object) {

        // null不可被序列化
        if (object == null) {
            return false;
        }

        Class clazz = object.getClass();

        // 如果有JsonSerializable注解，即可被序列化
        return clazz.isAnnotationPresent(JsonSerializable.class);
    }
}
```

# 使用自定义注解

首先创建一个`BookModel`类：

```java
/**
 * 书籍信息
 */
@JsonSerializable
public class BookModel {

    /**
     * 书名
     */
    @JsonElement(key = "bookname") // 在JSON中将bookName重命名为bookname
    private String bookName;

    /**
     * 分类
     */
    @JsonElement
    private String category;

    /**
     * 价格
     */
    @JsonElement
    private int price;

    @Override
    public String toString() {
        return "BookModel{" +
                "bookName='" + bookName + '\'' +
                ", category='" + category + '\'' +
                ", price=" + price +
                '}';
    }

    /**
     * getter，setter和构造方法略
     */
}
```

接下来在main方法里构造对象，并将其序列化成JSON：

```java
public static void main(String[] args) {

    BookModel book = new BookModel("Head First Java", "Java", 55);

    try {
        System.out.println(JsonUtils.serializeToJson(book));
    } catch (InvocationTargetException | IllegalAccessException | NoSuchMethodException e) {
        e.printStackTrace();
    }
}
```

序列化后的结果将是这样：

```json
{
    "bookname":"Head First Java",
    "category":"Java",
    "price":"55"
}
```
