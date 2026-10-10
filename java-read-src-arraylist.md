---
title: "Java源码阅读 - ArrayList"
source: https://www.boris1993.com/java-read-src-arraylist.html
date: 2019-03-22
updated: 2026-10-10
tags: [Java, ArrayList]
categories: [学知识]
---

# Java源码阅读 - ArrayList

做技术，不能只知其然而不知其所以然。在知道了工具的原理之后，才能更高效的使用这个工具。在程序的世界里，源码里面没有秘密，看懂了源码，也就看懂了原理。

这次就来阅读一下`ArrayList`的源码。

## ArrayList的特性

`ArrayList`有如下几个特性：

-   底层是一个动态扩容的数组
-   它允许存放多个`null`元素
-   允许存放多个重复的元素
-   元素在List中的顺序由添加顺序决定
-   不是线程安全的

## 类的声明

```java
public class ArrayList<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable { ... }
```

上面代码声明了一个叫`ArrayList`的泛型类，继承了`AbstractList`，并实现了`List`，`RandomAccess`，`Cloneable`，`Serializable`接口。

`AbstractList`抽象类提供了一个“骨架”级别的`List`接口的实现，用来减少实现一个支持随机存储的`List`的工作量。

`RandomAccess`中没有声明任何方法，是一个标记接口(marker interface)，表明了这个类支持快速(通常是`O(1)`时间复杂度)的随机存取。在遍历一个集合前，可以用`instanceof`判断这个集合是否实现了`RandomAccess`，来选择合适的遍历方法。

`Cloneable`也是一个标记接口，表明了这个类允许使用`Object.clone()`命令进行属性到属性的复制。

`Serializable`也是一个标记接口，表明在这个类上启用Java的序列化功能。

## 如何存储数据

```java
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * The size of the ArrayList (the number of elements it contains).
 *
 * @serial
 */
private int size;
```

`elementData`数组用来实际存放数据，`ArrayList`的空间(capacity)对应这个数组的长度(size)。`ArrayList`实现了自己的序列化(`ArrayList#writeObject()`)和反序列化(`ArrayList#readObject()`)方法，所以加上`transient`关键字来使`elementData`不参与Java自带的序列化和反序列化过程。

`size`成员变量记录当前`ArrayList`中元素的数量。

## 构造方法

`ArrayList`有三个构造方法

-   使用默认大小的`ArrayList()`
-   指定最初大小的`ArrayList(int initialCapacity)`
-   根据一个给定集合来初始化的`ArrayList(Collection<? extends E> c)`

### 使用默认大小

类中首先指定了默认的大小

```java
/**
 * Default initial capacity.
 */
private static final int DEFAULT_CAPACITY = 10;
```

但是，在它下面，还有这么一个东西：

```java
/**
 * Shared empty array instance used for default sized empty instances. We
 * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
 * first element is added.
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

在最初被构造时，`elementData`会先指向`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，而不是直接创建一个容量为10的数组。

```java
/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

这样做的好处在于可以更合理的利用空间。试想一下，如果某个场景中需要创建5个`ArrayList`备用，如果直接就分配好空间的话，那么就会消耗掉至少50个元素所需要的空间。所以Java选择先将`elementData`指向一个空数组，在向`ArrayList`中添加数据时，再去创建合适大小的数组。

### 指定最初大小

```java
/**
 * Constructs an empty list with the specified initial capacity.
 *
 * @param  initialCapacity  the initial capacity of the list
 * @throws IllegalArgumentException if the specified initial capacity
 *         is negative
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

当指定的大小是一个正整数时，Java会创建好对应大小的数组，并将`elementData`指向这个数组；如果指定的大小为零，那么Java也会将`elementData`指向一个共享的空数组`EMPTY_ELEMENTDATA`，注意这个空数组与上文提到的不是同一个；如果指定的大小为负数，则抛出一个异常。

那么为什么要专门把`EMPTY_ELEMENTDATA`和`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`区分出来呢？`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`的JavaDoc是这么说的：

> We distinguish this from `EMPTY_ELEMENTDATA` to know how much to inflate when first element is added.  
> 我们将它与`EMPTY_ELEMENTDATA`区分开来，是方便在添加第一个元素时计算要扩张多少空间。

### 根据给定的集合初始化

```java
/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 *
 * @param c the collection whose elements are to be placed into this list
 * @throws NullPointerException if the specified collection is null
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

程序首先试图调用给定集合的`Collection#toArray()`方法，将集合转换成一个`Object[]`数组。

当数组中有元素时，检查`elementData`的数据类型是否为`Object[]`类型，如果不是则使用`Arrays.copyOf()`方法重新复制元素到一个`Object[]`对象中；而当数组中没有元素时，则重新使`elementData`指向`EMPTY_ELEMENTDATA`。

## 添加元素

当添加元素时，首先会调用`ensureCapacityInternal()`方法，来保证空间足够。保证有足够空间后，就会向`elementData[size]`处放置被添加的元素，并且使`size`加一。

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

## 扩容

`ensureCapacityInternal()`方法用于确保在添加元素时有足够的空间。如果空间不足，则会调用`grow()`方法扩容。

`grow()`方法会将`elementData`扩张为当前的1.5倍空间，并使用`Arrays.copyOf()`方法将元素放入新的数组。

```java
/**
 * 确保空间
 */
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

/**
 * 计算扩容目标
 */
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    // 检查目标容量是否大于当前已有容量
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

/**
 * Increases the capacity to ensure that it can hold at least the
 * number of elements specified by the minimum capacity argument.
 *
 * 增加容量，以确保至少可以容纳minCapacity所指定个数的元素
 *
 * @param minCapacity the desired minimum capacity 目标最小容量
 */
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;

    // newCapacity = olcCapacity + (oldCapacity / 2)
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

## 删除元素

`ArrayList`提供了两种方式来删除一个元素：根据元素位置(index)删除，和匹配元素删除。

### 根据位置删除

根据位置删除时，首先会检查给定的位置是否越界。如果没有越界，就会先取出被删除的元素，用来向调用方返回。

删除元素的方法是将`index+1`后面的元素重新放在`index`起始的位置上。可以看出，删除操作的消耗是比较高的。

在重新排列元素后，数组中最后一个元素将与倒数第二个元素重复。所以还需要将最后一个元素置为null，并将size减一。

```java
/**
 * Removes the element at the specified position in this list.
 * Shifts any subsequent elements to the left (subtracts one from their
 * indices).
 *
 * @param index the index of the element to be removed
 * @return the element that was removed from the list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);

    // 计算要移动的元素数量
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(
            // 源
            elementData,
            // 源位置
            index+1,
            // 目标
            elementData,
            // 目标位置
            index,
            // 要复制的个数
            numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```

### 匹配元素删除

如果向`remove()`方法提供了一个对象，那么`ArrayList`会遍历`elementData`，并会删除第一个与给定对象匹配的元素。

```java
/**
 * Removes the first occurrence of the specified element from this list,
 * if it is present.  If the list does not contain the element, it is
 * unchanged.  More formally, removes the element with the lowest index
 * <tt>i</tt> such that
 * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
 * (if such an element exists).  Returns <tt>true</tt> if this list
 * contained the specified element (or equivalently, if this list
 * changed as a result of the call).
 *
 * @param o element to be removed from this list, if present
 * @return <tt>true</tt> if this list contained the specified element
 */
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

/*
 * Private remove method that skips bounds checking and does not
 * return the value removed.
 */
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

## 缩减容量

`ArrayList#trimToSize()`方法可以将`ArrayList`的容量缩减至当前元素个数。这个操作需要通过`Arrays.copyOf()`方法进行，所以成本也是比较高的。

```java
/**
 * Trims the capacity of this <tt>ArrayList</tt> instance to be the
 * list's current size.  An application can use this operation to minimize
 * the storage of an <tt>ArrayList</tt> instance.
 */
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

## Fail fast

在会改变`elementData`大小的方法中，经常会看到类似`modCount++`这样的操作。那么这个操作的目的是什么呢？

首先来看看`modCount`成员变量的JavaDoc是怎么说的。

```java
/**
 * The number of times this list has been <i>structurally modified</i>.
 * Structural modifications are those that change the size of the
 * list, or otherwise perturb it in such a fashion that iterations in
 * progress may yield incorrect results.
 *
 * <p>This field is used by the iterator and list iterator implementation
 * returned by the {@code iterator} and {@code listIterator} methods.
 * If the value of this field changes unexpectedly, the iterator (or list
 * iterator) will throw a {@code ConcurrentModificationException} in
 * response to the {@code next}, {@code remove}, {@code previous},
 * {@code set} or {@code add} operations.  This provides
 * <i>fail-fast</i> behavior, rather than non-deterministic behavior in
 * the face of concurrent modification during iteration.
 *
 * <p><b>Use of this field by subclasses is optional.</b> If a subclass
 * wishes to provide fail-fast iterators (and list iterators), then it
 * merely has to increment this field in its {@code add(int, E)} and
 * {@code remove(int)} methods (and any other methods that it overrides
 * that result in structural modifications to the list).  A single call to
 * {@code add(int, E)} or {@code remove(int)} must add no more than
 * one to this field, or the iterators (and list iterators) will throw
 * bogus {@code ConcurrentModificationExceptions}.  If an implementation
 * does not wish to provide fail-fast iterators, this field may be
 * ignored.
 */
protected transient int modCount = 0;
```

也就是说，`modCount`记录了一个`List`的结构被修改的次数，并且提到了如果在迭代过程中修改了`List`的结构，那么可能会导致得到错误的结果。

在迭代或者序列化的过程中，程序会检查`modCount`的值是否被修改过，如果被修改，就会抛出`ConcurrentModificationException`异常。比如`ArrayList.Itr#next()`方法：

```java
@SuppressWarnings("unchecked")
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}

final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

## 序列化与反序列化

如上文所说，`ArrayList`实现了自己的序列化与反序列化方法，所以`elementData`使用`transient`修饰。

在序列化时，程序并不是直接序列化`elementData`这个数组，而是只取出数组中有效的元素(包括null元素)，并逐个序列化每个元素的对象。

```java
/**
 * Save the state of the <tt>ArrayList</tt> instance to a stream (that
 * is, serialize it).
 *
 * @serialData The length of the array backing the <tt>ArrayList</tt>
 *             instance is emitted (int), followed by all of its elements
 *             (each an <tt>Object</tt>) in the proper order.
 */
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();
    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);
    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

在反序列化时，首先会使`elementData`指向`EMPTY_ELEMENTDATA`，只在有元素会被反序列化时，才会为`elementData`扩容并逐个反序列化对应的对象。

```java
/**
 * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
 * deserialize it).
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;
    // Read in size, and any hidden stuff
    s.defaultReadObject();
    // Read in capacity
    s.readInt(); // ignored
    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        int capacity = calculateCapacity(elementData, size);
        SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
        ensureCapacityInternal(size);
        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
```

## 迭代

迭代器是用于遍历各个集合的标准方法，`ArrayList`也不例外，它提供了通过`iterator()`方法获得的`Iterator`的迭代器，以及通过`listIterator()`方法获得的`ListItr`迭代器。

### Iterator迭代器

迭代器`Iterator`接口提供了四个方法：

```java
public interface Iterator<E> {
    // 检查是否还有下一个元素
    boolean hasNext();

    // 获取下一个元素
    E next();

    // 删除一个当前的元素
    void remove();

    // 使用Lambda表达式进行遍历
    forEachRemaining(Consumer<? super E> action);
}
```

在调用`ArrayList#iterator()`方法后，我们可以得到一个`Itr`内部类的实例。

```java
/**
 * An optimized version of AbstractList.Itr
 */
private class Itr implements Iterator<E> {
    // 下一个将被返回的元素的下标
    int cursor;

    // 上一个被返回的元素的下标; -1 代表尚没有元素被返回
    int lastRet = -1;

    // 在开始迭代时这个`ArrayList`的`modCount`
    // 在迭代操作时，迭代器将反复检查`expectedModCount`与当前`modCount`是否相同，一旦不同就会触发`fail fast`机制。
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        // 如果当前迭代位置的下标等于集合的长度
        // 则说明迭代已经结束
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        // 检查集合的结构是否在迭代过程中被修改过
        checkForComodification();

        // 取得当前迭代位置的下标
        int i = cursor;
        // 如果下标超出了集合的长度
        // 则抛出异常
        if (i >= size)
            throw new NoSuchElementException();

        // 取得该集合的所有元素
        Object[] elementData = ArrayList.this.elementData;

        // 如果当前位置超出了集合的长度
        // 则说明集合的结构发生了变化
        if (i >= elementData.length)
            throw new ConcurrentModificationException();

        // 将游标向前移动一个位置
        cursor = i + 1;

        // 将上次返回的下标指向当前位置，并返回该位置的元素
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        // 因为该方法删除的是lastRet指向的元素
        // 而在未调用next方法前，lastRet是-1
        // 所以不允许这样操作
        if (lastRet < 0)
            throw new IllegalStateException();

        // 检查集合的结构是否在迭代过程中被修改过
        checkForComodification();

        try {
            // 调用集合的remove方法删除元素
            ArrayList.this.remove(lastRet);
            // 因为集合少了一个元素，所以将游标向前移动一个位置
            cursor = lastRet;
            // 而上次被返回的元素已经没了，
            // 所以lastRet指向-1
            lastRet = -1;
            // 因为ArrayList#remove修改了modCount
            // 所以这里同步expectedModCount，确保后续的迭代过程中不会触发fail fast机制
            // 所以使用迭代器删除元素是安全的
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            // 如果在删除操作时发生了数组越界
            // 则说明集合的结构被改变了
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

### ListItr迭代器

`ArrayList`提供了两个方法用来获得一个`ListItr`迭代器，其区别是有无指定下标。

```java
/**
 * Returns a list iterator over the elements in this list (in proper
 * sequence), starting at the specified position in the list.
 * The specified index indicates the first element that would be
 * returned by an initial call to {@link ListIterator#next next}.
 * An initial call to {@link ListIterator#previous previous} would
 * return the element with the specified index minus one.
 *
 * <p>The returned list iterator is <a href="#fail-fast"><i>fail-fast</i></a>.
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

/**
 * Returns a list iterator over the elements in this list (in proper
 * sequence).
 *
 * <p>The returned list iterator is <a href="#fail-fast"><i>fail-fast</i></a>.
 *
 * @see #listIterator(int)
 */
public ListIterator<E> listIterator() {
    return new ListItr(0);
}
```

`ListItr`继承了前面提到的`Itr`类，也就是说它拥有`Itr`类的所有方法。同时它实现了`ListIterator`接口。

`ListIterator`接口继承了`Iterator`接口。除了`Iterator`接口中定义的方法外，`ListIterator`还另外声明了数个方法。

```java
public interface ListIterator<E> extends Iterator<E> {
    // 在Iterator中存在的方法这里就略过了

    // 检查是否有前一个元素
    boolean hasPrevious();

    // 向前迭代一个元素
    E previous();

    // 获取next()方法将要迭代到的元素的下标
    int nextIndex();

    // 获取previous()方法将要迭代到的元素的下标
    int previousIndex();

    // 把通过next()或者previous()得到的元素修改为传入的数据
    void set(E e);

    // 将一个新的元素插入到上一个被迭代的元素和下一个被迭代的元素之间
    void add(E e);
}
```

`ArrayList`中具体的`ListItr`的实现是这样子的：

```java
/**
 * An optimized version of AbstractList.ListItr
 */
private class ListItr extends Itr implements ListIterator<E> {
    // 在构造时直接将cursor指向指定的下标
    ListItr(int index) {
        super();
        cursor = index;
    }

    // 因为数组首个元素的下标是0，而cursor不会是负数
    // 所以当cursor不为0时，前面就有元素可供迭代
    public boolean hasPrevious() {
        return cursor != 0;
    }

    // cursor指向下一个被迭代的元素的下标
    public int nextIndex() {
        return cursor;
    }

    // cursor减一就得到了前一个元素的下标
    public int previousIndex() {
        return cursor - 1;
    }

    // 向前迭代一个元素
    @SuppressWarnings("unchecked")
    public E previous() {
        // 检查集合的结构有没有被修改过
        checkForComodification();

        // 取得前一个元素的下标
        int i = cursor - 1;

        // 如果前一个元素的下标是负数
        // 则抛出NoSuchElementException
        if (i < 0)
            throw new NoSuchElementException();

        // 取到集合的所有元素
        Object[] elementData = ArrayList.this.elementData;

        // 如果此时下标超出了所有元素的长度
        // 则认定集合的结构被外部改变过了
        if (i >= elementData.length)
            throw new ConcurrentModificationException();

        // 将cursor指向前一个元素
        cursor = i;

        // 取得前一个元素，并重新设定lastRet的值
        return (E) elementData[lastRet = i];
    }

    // 将上一个迭代到的元素修改为指定的数据
    public void set(E e) {
        // 尚无元素被迭代到的时候
        // lastRet是-1
        // 此时没有元素可以被修改
        if (lastRet < 0)
            throw new IllegalStateException();

        // 检查集合的结构有没有被修改过
        checkForComodification();

        try {
            // 尝试使用`ArrayList#set`方法修改上一个被迭代位置的值
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            // 如果发生下标越界，则认定集合的结构被外部修改过
            throw new ConcurrentModificationException();
        }
    }

    // 在上一个元素与下一个元素之间插入数据
    public void add(E e) {
        // 检查集合的结构有没有被修改过
        checkForComodification();

        try {
            // 取得当前迭代的位置
            int i = cursor;

            // 尝试在当前位置插入一个元素
            ArrayList.this.add(i, e);

            // 将cursor后移一个位置
            cursor = i + 1;

            lastRet = -1;

            // 因为修改了集合结构后，modCount会改变
            // 所以需要同步expectedModCount的值
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            // 如果在操作过程中发生了下标越界
            // 则认定集合的结构被外部修改了
            throw new ConcurrentModificationException();
        }
    }
}
```
