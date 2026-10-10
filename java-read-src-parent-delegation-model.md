---
title: "Java源码阅读 - 类加载的双亲委派模型"
source: https://www.boris1993.com/java-read-src-parent-delegation-model.html
date: 2023-12-13
updated: 2026-10-10
tags: [Java, 双亲委派, 类加载]
categories: [学知识]
---

# Java源码阅读 - 类加载的双亲委派模型

关于Java的类加载机制，尽管我看过几篇文章，知道个`双亲委派模型`，但是从来没钻进源码里看它到底是怎么委派的。

## 什么双亲？怎么委派？

我刚一开始听到`双亲委派`，还纳闷咋还双亲？后来才知道，这纯纯就是`Parent Delegation`这个词的误译。`Parent`这里指的并不是双亲，而是指父辈。所以看到有人翻译为“向上委托模型”，我感觉这个翻译更好一点，至于另一个翻译“啃老模型”……倒也没毛病……

至于怎么委托，相信各位都背的滚瓜烂熟了。那就是，当类加载器收到类加载请求的时候，它首先会把这个请求委托给上一层的类加载器去尝试加载，直到委托到启动类加载器；只有当上一层类加载器无法完成这个加载请求的时候，次一级类加载器才会尝试自己加载。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/classloader-relationship.png)

## 代码上的实现

截图里面的代码就是`ClassLoader#loadClass`方法的实现，来自`Liberica JDK 8`。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/load-class-method-implementation.png)

看得出来，逻辑还是很简单易懂的。一进来先加个锁，防止出现并发问题。然后检查这个类是不是已经被加载了。没被加载的话，就一层层向上委托，直到到达启动类加载器。如果上一层类加载器返回了`null`或者抛出了`ClassNotFoundException`异常，就说明它没找到这个类，那么本层类加载器就会尝试加载这个类，如果找不到的话，它就接着把请求交回下一层的类加载器。

虽然上面的图和代码已经可以解释双亲委派的工作机制，但我还是喜欢调试进去看看代码具体是咋走的。所以我写了这么几行，用来调试类加载器。

```java
public class Main {
    public static void main(String[] args) throws ClassNotFoundException {
        ClassLoader classLoader = Main.class.getClassLoader();
        System.out.println(classLoader.toString());
        // 在下面这一行下个断点，走到这一行之后再给 ClassLoader#loadClass 下个断点
        classLoader.loadClass("DaemonThreadDemo");
        classLoader.loadClass("java.util.HashMap");
    }
}
```

断点从`classLoader.loadClass("DaemonThreadDemo")`这一行进去并停留在`if (c == null)`之后可以看到，目前类加载的委托请求是交给`AppClassLoader`，看得出来，这个就是应用类加载器。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/debug-classloader-1.png)

继续往下走到`c = parent.loadClass(name, false)`这一行，然后给`parent`变量添加一个监视，就可以看到接下来`AppClassLoader`要把这个类加载请求委托给`ExtClassLoader`，同理可得，这个就是扩展类加载器。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/debug-classloader-2.png)

接着往下走，继续调试`ExtClassLoader`，这时候可以看到`parent`是`null`。没有了`parent`，这个类加载器就会将这个类加载请求委托给启动类加载器并尝试加载这个类。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/debug-classloader-3.png)

逐层点进去，可以看到如下代码：

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/debug-classloader-4.png)

emmmm……走到了一个native方法了呢……嘛，里面的代码先不管了，看名字能猜得出来，在这里会调用启动类加载器来尝试加载这个类。

因为要加载的`DaemonThreadDemo`类并不归启动类加载器管，所以`findBootstrapClassOrNull`返回了`null`。`ExtClassLoader`得知启动类加载器加载失败，那么它自己就会再尝试加载。然而这个类也不归扩展类加载器管，所以在`ExtClassLoader`里面调用`findClass`方法会抛出`ClassNotFoundException`异常并返回到`AppClassLoader`。

这时候，因为`DaemonThreadDemo`这个类归应用类加载器管，所以这一次调用`findClass`成功的找到了这个类。

![](https://blog-static.boris1993.com/java-read-src-parent-delegation-model/debug-classloader-5.png)

所以代码可以成功走到`return c`这一行，来完成一个类的加载。

## 破坏双亲委派模型

说到双亲委派模型，必会谈到怎么破坏它。看完上面的代码就明白了，我们可以自己创建一个自定义类加载器，并重写`loadClass`方法，不让它向上委派就行了。

## 番外：尝试理解`findBootstrapClass`

虽然这部分是C和C++的实现，但还是想硬着头皮尝试看一下。到`Bellsoft`的官网下载虚拟机的源码之后，我找到了`FindBootStrapClass`函数的实现：

```c
// 这部分代码在 java_md.c

/*
 * The implementation for finding classes from the bootstrap
 * class loader, refer to java.h
 */
static FindClassFromBootLoader_t *findBootClass = NULL;

jclass FindBootStrapClass(JNIEnv *env, const char *classname)
{
   HMODULE hJvm;

   if (findBootClass == NULL) {
       hJvm = GetModuleHandle(JVM_DLL);
       if (hJvm == NULL) return NULL;
       /* need to use the demangled entry point */
       findBootClass = (FindClassFromBootLoader_t *)GetProcAddress(hJvm,
            "JVM_FindClassFromBootLoader");
       if (findBootClass == NULL) {
          JLI_ReportErrorMessage(DLL_ERROR4, "JVM_FindClassFromBootLoader");
          return NULL;
       }
   }
   return findBootClass(env, classname);
}
```

这个C语言……确实跟我大学学的C语言不一样啊……爬了些文，大概理解这里是要找`JVM_FindClassFromBootLoader`这个函数的实际地址，然后赋给`findBootClass`指针并执行它的代码。于是我接着挖到了`JVM_FindClassFromBootLoader`函数的实现。

```cpp
// 这部分代码在 jvm.cpp

// Returns a class loaded by the bootstrap class loader; or null
// if not found.  ClassNotFoundException is not thrown.
//
// Rationale behind JVM_FindClassFromBootLoader
// a> JVM_FindClassFromClassLoader was never exported in the export tables.
// b> because of (a) java.dll has a direct dependecy on the  unexported
//    private symbol "_JVM_FindClassFromClassLoader@20".
// c> the launcher cannot use the private symbol as it dynamically opens
//    the entry point, so if something changes, the launcher will fail
//    unexpectedly at runtime, it is safest for the launcher to dlopen a
//    stable exported interface.
// d> re-exporting JVM_FindClassFromClassLoader as public, will cause its
//    signature to change from _JVM_FindClassFromClassLoader@20 to
//    JVM_FindClassFromClassLoader and will not be backward compatible
//    with older JDKs.
// Thus a public/stable exported entry point is the right solution,
// public here means public in linker semantics, and is exported only
// to the JDK, and is not intended to be a public API.

JVM_ENTRY(jclass, JVM_FindClassFromBootLoader(JNIEnv* env,
                                              const char* name))
  JVMWrapper2("JVM_FindClassFromBootLoader %s", name);

  // Java libraries should ensure that name is never null...
  if (name == NULL || (int)strlen(name) > Symbol::max_length()) {
    // It's impossible to create this class;  the name cannot fit
    // into the constant pool.
    return NULL;
  }

  TempNewSymbol h_name = SymbolTable::new_symbol(name, CHECK_NULL);
  Klass* k = SystemDictionary::resolve_or_null(h_name, CHECK_NULL);
  if (k == NULL) {
    return NULL;
  }

  if (TraceClassResolution) {
    trace_class_resolution(k);
  }
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END
```

好吧，更看不懂了。继续爬了文之后，理解这里大致做了三件事：从常量池中拿到类名的信息；查找类的信息并实例化`Klass`；将`Klass`对象转换为`jclass`类型并返回。

算了，虚拟机源码就看到这吧……有兴趣的同志可以看JVM系列(四)：java方法的查找过程实现这篇博客。
