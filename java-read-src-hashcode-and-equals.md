---
title: "Java源码阅读 - equals和hashCode方法"
source: https://www.boris1993.com/java-read-src-hashcode-and-equals.html
date: 2020-01-06
updated: 2026-10-10
tags: [Java, LinkedList]
categories: [学知识]
---

# Java源码阅读 - equals和hashCode方法

做技术，不能只知其然而不知其所以然。在知道了工具的原理之后，才能更高效的使用这个工具。在程序的世界里，源码里面没有秘密，看懂了源码，也就看懂了原理。

这次就来阅读一下`Object`类里面`hashCode`方法和`equals`方法的源码。

## 先看看代码

```java
public native int hashCode();

public boolean equals(Object obj) {
    return (this == obj);
}
```

可以看出，`hashCode`方法是一个native方法，`equals`方法比较了两个对象是否指向同一个内存的地址。

## hashCode

### 什么是hash

要搞清楚`hashCode`干了什么，那就得要知道`hash`是什么。

> 散列函数（英语：Hash function）又称散列算法、哈希函数，是一种从任何一种数据中创建小的数字“指纹”的方法。散列函数把消息或数据压缩成摘要，使得数据量变小，将数据的格式固定下来。该函数将数据打乱混合，重新创建一个叫做散列值（hash values，hash codes，hash sums，或hashes）的指纹。散列值通常用一个短的随机字母和数字组成的字符串来代表。好的散列函数在输入域中很少出现散列冲突。在散列表和数据处理中，不抑制冲突来区别数据，会使得数据库记录更难找到。

### Java中的hashCode方法

在`Object`类中的`hashCode`方法是一个native方法，我们没办法直接得知它的实现方式，但是我们依旧可以从它的JavaDoc中得知一些信息。

```java
/**
     * Returns a hash code value for the object. This method is
     * supported for the benefit of hash tables such as those provided by
     * {@link java.util.HashMap}.
     * <p>
     * The general contract of {@code hashCode} is:

     * <p>
     * As much as is reasonably practical, the hashCode method defined by
     * class {@code Object} does return distinct integers for distinct
     * objects. (This is typically implemented by converting the internal
     * address of the object into an integer, but this implementation
     * technique is not required by the
     * Java&trade; programming language.)
     *
     * @return  a hash code value for this object.
     * @see     java.lang.Object#equals(java.lang.Object)
     * @see     java.lang.System#identityHashCode
     */
    public native int hashCode();
```

上面花了很大的篇幅讲了如果要重新实现`hashCode`方法所需要遵循的原则，但是很可惜，我们现在暂时不关注这些。我们现在关注的，是最后一段的内容。

在最后一段中，它讲了通常情况下，程序是怎样计算出`hashCode`的值的。

> This is typically implemented by converting the internal address of the object into an integer  
> 通常来说，这是通过把内部的地址转换成一个整型数来实现的

当然，并不是所有的类都使用了这个计算方法，比如`String`就重新实现了自己的`hashCode`方法：

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

## equals

`equals`方法的作用是比较两个对象的内容是否相同。一般来说，`Object`类中提供的`equals`方法是没办法满足各个类型自己的需要的，所以它们基本上都实现了自己的`equals`方法。

用一个简单的例子来讲：

```java
String str1 = "aaa";
String str2 = "aaa";
str1.equals(str2); // true
```

显然，`str1`和`str2`是两个不同的对象，如果直接比较它们的内存地址，那么得到的结果肯定是false。所以可以肯定的是，`String`类重写了`equals`方法。那么，我们就简单看一下`String`是怎样实现`equals`方法的：

```java
public boolean equals(Object anObject) {
    // 先检查两个对象的地址是否相同
    if (this == anObject) {
        return true;
    }

    // 如果被比较的对象地址不同，但它类型相同
    // 那么继续进行比较
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;

        // 如果被比较的字符串与本字符串长度相同
        // 那么继续比较其中char数组中的每个元素是否相同
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

既然每个类型都可以实现自己的`equals`方法，那么必然有一个规则来约束它们的实现方式，以保证在何时何地`equals`都可以得到合理的结果。

在`equals`方法的JavaDoc中描述了重写该方法所需要遵守的规则：

> It is *reflexive*: for any non-null reference value `x`, `x.equals(x)` should return `true`.  
> It is *symmetric*: for any non-null reference values `x` and `y`, `x.equals(y)` should return `true` if and only if `y.equals(x)` returns `true`.  
> It is *transitive*: for any non-null reference values `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`.  
> It is *consistent*: for any non-null reference values `x` and `y`, multiple invocations of `x.equals(y)` consistently return `true` or consistently return `false`, provided no information used in `equals` comparisons on the objects is modified.  
> For any non-null reference value `x`, `x.equals(null)` should return `false`.

翻译过来就是：

> 自反性：对于一个非null的引用值，`x.equals(x)`应当返回`true`。  
> 对称性：对于两个非null的引用值`x`和`y`，当且仅当`y.equals(x)`时，`x.equals(y)`返回`true`。  
> 传递性：对于任意非null的引用值`x`，`y`和`z`，如果`x.equals(y)`返回`true`，而且`y.equals(z)`返回`true`，那么`x.equals(z)`也应返回`true`。  
> 一致性：对于任意非null的引用值`x`和`y`，当两者都未被修改时，多次调用`x.equals(y)`都应一直返回`true`或者`false`。  
> 对于任意非null的引用值`x`，`x.equals(null)`应返回`false`。

## hashCode方法与equals方法的关系

在`equals`方法的JavaDoc上有这么一段话：

> Note that it is generally necessary to override the `hashCode` method whenever this method is overridden, so as to maintain the general contract for the `hashCode` method, which states that equal objects must have equal hash codes.  
> 在重写`equals`方法时，通常也需要一并重写`hashCode`方法，以便维护`hashCode`方法的约定，即相等的对象必须拥有相同的哈希码

而在`hashCode`方法的JavaDoc中，它有着这样的实现约定：

> Whenever it is invoked on the same object more than once during an execution of a Java application, the `hashCode` method must consistently return the same integer, provided no information used in `equals` comparisons on the object is modified. This integer need not remain consistent from one execution of an application to another execution of the same application.
> 
> If two objects are equal according to the `equals(Object)` method, then calling the `hashCode` method on each of the two objects must produce the same integer result.
> 
> It is *not* required that if two objects are unequal according to the `java.lang.Object#equals(java.lang.Object)` method, then calling the `hashCode` method on each of the two objects must produce distinct integer results. However, the programmer should be aware that producing distinct integer results for unequal objects may improve the performance of hash tables.

即：

> 在程序运行过程中，不论`hashCode`方法被调用了多少次，其返回结果都必须是一个恒定的整型值，以表明在使用`equals`比较对象时所需的信息没有被修改过。但是在程序每次运行之间，`hashCode`返回的值则不需要保持一致
> 
> 如果两个对象使用`equals`方法比较得出了相同(equal)的结论，那么对这两个对象执行`hashCode`方法得到的值也必须相同
> 
> 在两个对象使用`equals`方法比较得出了不相同(not equal)的结论时，对这两个对象执行`hashCode`方法得到的值却可以相同。然而，开发人员需要意识到，给不同的对象返回不同的哈希码可以提升hash table的性能

综上所述，我们可以得出如下结论：

-   两个相同(equal)的对象必须拥有相同的哈希码
-   两个哈希码相同的对象却不一定相同(equal)

那么，这两条结论会对我们的程序造成什么影响呢？

首先我们看一下第一条。以`Set`举例，`Set`会根据对象的`hashCode`来寻找对象的存储位置，那么可想而知，如果两个对象的值相同，哈希码却不同，那么就会导致`Set`中出现多个重复数据的情况。

而第二条结论出现的原因则是，目前没有任何一种哈希算法，可以保证对每个传入的值都可以计算出一个不同的哈希，这种情况的学名叫`哈希碰撞`，所以我们只能尽可能的减少出现哈希碰撞的可能性。至于Java如何应对哈希碰撞，我将在后续的博文中进行解释。
