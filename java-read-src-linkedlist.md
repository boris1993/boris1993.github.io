---
title: "Java源码阅读 - LinkedList"
source: https://www.boris1993.com/java-read-src-linkedlist.html
date: 2019-12-25
updated: 2026-10-10
tags: [Java, LinkedList]
categories: [学知识]
---

# Java源码阅读 - LinkedList

做技术，不能只知其然而不知其所以然。在知道了工具的原理之后，才能更高效的使用这个工具。在程序的世界里，源码里面没有秘密，看懂了源码，也就看懂了原理。

这次就来阅读一下`LinkedList`的源码。

## LinkedList的特性

`LinkedList`有如下几个特性：

-   底层的数据结构是双向链表
-   存储的数据允许为`null`
-   允许存放重复的数据
-   元素在List中的顺序由添加顺序决定
-   不是线程安全的

## 类的声明

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

上面代码声明了一个名为`LinkedList`的泛型类，继承了`AbstractSequentialList`，并实现了`List`，`Deque`，`Cloneable`，`Serializable`接口。

`AbstractSequentialList`抽象类提供了一个“骨架”级别的List实现，用来减少实现一个支持顺序读写的`List`的工作量。

`Deque`接口约定了要实现一个双向队列(Double Ended Queue)所必须要实现的方法。

`Cloneable`是一个标记接口，表明了这个类允许使用`Object.clone()`命令进行属性到属性的复制。

`Serializable`也是一个标记接口，表明在这个类上启用Java的序列化功能。

## 如何存储数据

```java
transient int size = 0;

/**
    * Pointer to first node.
    * Invariant: (first == null && last == null) ||
    *            (first.prev == null && first.item != null)
    */
transient Node<E> first;

/**
    * Pointer to last node.
    * Invariant: (first == null && last == null) ||
    *            (last.next == null && last.item != null)
    */
transient Node<E> last;
```

`first`和`last`是两个`Node`对象，分别指向了链表中的第一个节点和最后一个节点。`size`保存了这个链表中元素的个数。

`Node`类是`LinkedList`类中的一个内部类，它定义了一个元素实际上是如何被存储的。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

`item`是实际存储的数据，`next`和`prev`则分别指向了下一个元素和上一个元素。

## 构造方法

`LinkedList`有两个构造方法，分别用来初始化一个空的链表，和从一个给定的集合中取出元素来初始化一个链表。

### 无参构造方法

```java
/**
    * Constructs an empty list.
    */
public LinkedList() {
}
```

无参的构造方法实际上什么都没有做，返回的`LinkedList`对象中，`size`为默认值`0`，`first`和`last`的值都是`null`。

### 从集合初始化的构造方法

```java
/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 *
 * @param  c the collection whose elements are to be placed into this list
 * @throws NullPointerException if the specified collection is null
 */
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

这个构造方法首先创建了一个空的`LinkedList`，然后调用了`addAll`方法将集合中的数据放到这个链表中。

## 插入数据

在`LinkedList`中插入数据有三种方式：在头部增加节点、在尾部增加节点，和在某个元素间插入节点。

### 在头部增加节点

要在链表头部增加节点，可以使用`addFirst(E)`方法。

```java
/**
 * Inserts the specified element at the beginning of this list.
 *
 * @param e the element to add
 */
public void addFirst(E e) {
    linkFirst(e);
}
```

该方法又调用了一个`private`方法`linkFirst(E)`实现在头部插入数据。

```java
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

`linkFirst(E)`方法首先取出当前的头部元素`first`，然后构造了一个新的`Node`对象，新对象的`prev`值为`null`，代表它是一个头部元素，`next`值为原来的`first`，它存储的数据则是这次插入的数据。然后它将链表的`first`设为这次新增加的元素。

根据链表的特性可以知道，如果一个链表不是空的，那么它的`first`必定非空；反之，如果它的`first`为`null`，那么这个链表一定为空。所以根据这个规则，它会判断在插入元素前，这个链表是不是空的，如果是空的，那么新元素就同时作为链表的尾`last`；如果不是空的，那么就让原来的`first`的`prev`指向新插入的元素。这样操作之后，新元素与原`first`元素之间就出现了一个双向的引用，即完成了一个小的双向链表。

最后使链表的`size`加一，就完成了一次新增头元素的操作。

### 在尾部增加节点

在尾部增加节点，可以使用`add(E)`方法或`addLast(E)`方法。

```java
/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
public void addLast(E e) {
    linkLast(e);
}

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #addLast}.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

可以看出，两个方法都是通过一个`private`方法`linkLast(E)`实现的。

```java
/**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

在尾部插入数据的操作与在头部插入数据的操作类似，依旧是构造一个新的节点，使原来的`last`节点指向新节点，然后根据原链表是否为空执行后续操作。在这里就不多赘述了。

### 在中间增加节点

要在链表中间插入数据，可以使用`add(int, E)`方法。

```java
/**
 * Inserts the specified element at the specified position in this list.
 * Shifts the element currently at that position (if any) and any
 * subsequent elements to the right (adds one to their indices).
 *
 * @param index index at which the specified element is to be inserted
 * @param element element to be inserted
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

这个方法所做的操作，简单来说就是，将新的元素放到指定位置，并将原来处于这个位置的元素及其所有后续元素全部后移一个位置。

首先它调用了`checkPositionIndex(int)`方法，我们看看它干了什么。

```java
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

/**
 * Tells if the argument is the index of a valid position for an
 * iterator or an add operation.
 */
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

/**
 * Constructs an IndexOutOfBoundsException detail message.
 * Of the many possible refactorings of the error handling code,
 * this "outlining" performs best with both server and client VMs.
 */
private String outOfBoundsMsg(int index) {
    return "Index: "+index+", Size: "+size;
}
```

这个方法检查了用户输入的插入位置是不是一个合法的位置，规则就是插入位置必须大于等于0且小于等于最大位置。

通过检查之后，它继续判断插入的位置是不是链表的末尾，如果是末尾的话，就直接调用`linkLast(E)`在链表尾部新增一个元素，否则它会先取出现在位于插入位置的节点，然后调用`linkBefore(E, Node)`在链表中间插入元素。

```java
/**
 * Returns the (non-null) Node at the specified element index.
 */
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

/**
 * Inserts element e before non-null Node succ.
 */
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

`node(int)`方法巧妙的利用了二分法，根据元素所在的位置来决定是从链表头部还是从尾部开始查找节点。

`linkBefore(E, Node)`方法进行的操作，就像我们在书中学习到的一样，先让新节点建立起到左右两个节点的连接，然后让右边的节点连接到新插入的节点，最后更新链表的大小。

### 用集合批量增加节点

之前我们在`LinkedList`的构造方法中看到了一个`addAll(Collection)`方法，现在就来看看它干了什么。

```java
/**
 * Appends all of the elements in the specified collection to the end of
 * this list, in the order that they are returned by the specified
 * collection's iterator.  The behavior of this operation is undefined if
 * the specified collection is modified while the operation is in
 * progress.  (Note that this will occur if the specified collection is
 * this list, and it's nonempty.)
 *
 * @param c collection containing elements to be added to this list
 * @return {@code true} if this list changed as a result of the call
 * @throws NullPointerException if the specified collection is null
 */
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

/**
 * Inserts all of the elements in the specified collection into this
 * list, starting at the specified position.  Shifts the element
 * currently at that position (if any) and any subsequent elements to
 * the right (increases their indices).  The new elements will appear
 * in the list in the order that they are returned by the
 * specified collection's iterator.
 *
 * @param index index at which to insert the first element
 *              from the specified collection
 * @param c collection containing elements to be added to this list
 * @return {@code true} if this list changed as a result of the call
 * @throws IndexOutOfBoundsException {@inheritDoc}
 * @throws NullPointerException if the specified collection is null
 */
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```

这里的重头戏是`addAll(int, Collection)`方法，它实现了在链表中间批量插入节点的功能。`addAll(Collection)`实际上就是调用它在链表末尾批量插入节点。

首先`addAll`方法会检查插入的位置是否合法，如果不合法就会抛出`IndexOutOfBoundsException`异常。然后它将传入的集合转换成一个对象数组，并检查数组长度，如果长度是0，则说明链表内容未被改变，直接返回`false`。

然后它会检查插入的位置，并且记录下插入位置的上一个节点和下一个节点。

接下来这个方法开始遍历传入的集合，并将集合中的数据逐个插入到链表中。插入的逻辑与前面讲的类似，所以就不再赘述了。

最后它会完成一系列收尾工作，包括设定链表尾部的节点，和更新链表的长度，然后返回`true`，代表链表成功被更新了。

## 查询数据

因为`LinkedList`也是一个双向队列，所以它既允许从两端开始获取数据，又可以根据下标从指定位置获取数据。

### 取出头部的节点

`LinkedList`提供了多个方法来允许用户从链表头部取出数据，分别有：

```java
/**
 * Returns the first element in this list.
 *
 * @return the first element in this list
 * @throws NoSuchElementException if this list is empty
 */
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

/**
 * Retrieves, but does not remove, the head (first element) of this list.
 *
 * @return the head of this list, or {@code null} if this list is empty
 * @since 1.5
 */
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

/**
 * Retrieves and removes the head (first element) of this list.
 *
 * @return the head of this list, or {@code null} if this list is empty
 * @since 1.5
 */
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

从头部取出节点的操作都大同小异，区别只是在于取出空值之后是抛异常还是返回null，以及会不会同时删除头部元素。逻辑很简单，这里就不多赘述了。

除了上面列出的几个方法外，还有`pop()`、`pollFirst()`等方法也提供了相同的功能，但是代码内容大同小异，所以也不放上来了，以免浪费篇幅。

### 取出尾部的节点

`LinkedList`同样提供了数个方法用于从尾部取出节点，它们的逻辑也基本相同，这里同上文一样，仅展示部分代码。

```java
/**
 * Returns the last element in this list.
 *
 * @return the last element in this list
 * @throws NoSuchElementException if this list is empty
 */
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}

/**
 * Retrieves, but does not remove, the last element of this list,
 * or returns {@code null} if this list is empty.
 *
 * @return the last element of this list, or {@code null}
 *         if this list is empty
 * @since 1.6
 */
public E peekLast() {
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}

/**
 * Retrieves and removes the last element of this list,
 * or returns {@code null} if this list is empty.
 *
 * @return the last element of this list, or {@code null} if
 *     this list is empty
 * @since 1.6
 */
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}
```

### 取出中间的节点

要从链表中的某个位置取出节点，可以使用`get(int)`方法。

```java
/**
 * Returns the element at the specified position in this list.
 *
 * @param index index of the element to return
 * @return the element at the specified position in this list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

它首先还是检查了传入的下标是否合法，如果合法就调用`node(int)`方法取得该节点，并返回其数据。`node(int)`方法在上面已经介绍过，这里就不重复介绍了。

### 查询链表是否包含某个数据

`LinkedList`提供了`contains(Object)`方法用来查询该链表是否包含某个数据。

```java
/**
 * Returns {@code true} if this list contains the specified element.
 * More formally, returns {@code true} if and only if this list contains
 * at least one element {@code e} such that
 * <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>.
 *
 * @param o element whose presence in this list is to be tested
 * @return {@code true} if this list contains the specified element
 */
public boolean contains(Object o) {
    return indexOf(o) != -1;
}

/**
 * Returns the index of the first occurrence of the specified element
 * in this list, or -1 if this list does not contain the element.
 * More formally, returns the lowest index {@code i} such that
 * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
 * or -1 if there is no such index.
 *
 * @param o element to search for
 * @return the index of the first occurrence of the specified element in
 *         this list, or -1 if this list does not contain the element
 */
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

`contains(Object)`方法实际上是调用了`indexOf(Object)`方法，并检查其返回是否为`-1`，来判断这个值是否存在于该链表中。

`indexOf(Object)`方法的逻辑就是，从链表的头部开始，逐个检查其节点的值是否为传入的值。如果链表为空则直接返回`-1`。

## 修改数据

`LinkedList`提供了一个`set(int, E)`的方法用于修改某个节点的数据。

```java
/**
 * Replaces the element at the specified position in this list with the
 * specified element.
 *
 * @param index index of the element to replace
 * @param element element to be stored at the specified position
 * @return the element previously at the specified position
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

该方法首先检查传入的下标是否合法，检查通过后，它会为指定位置的节点设定新的数据，并返回该节点原有的数据。

## 删除数据

`LinkedList`提供了多个方法来从链表中删除节点。

```java
/**
 * Removes and returns the first element from this list.
 *
 * @return the first element from this list
 * @throws NoSuchElementException if this list is empty
 */
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

/**
 * Removes and returns the last element from this list.
 *
 * @return the last element from this list
 * @throws NoSuchElementException if this list is empty
 */
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
```

`removeFirst()`和`removeLast()`方法分别可以从链表的头部和尾部取出一个节点，并将其删除。如果链表是空的，则会抛出`NoSuchElementException`异常。

```java
/**
 * Removes the element at the specified position in this list.  Shifts any
 * subsequent elements to the left (subtracts one from their indices).
 * Returns the element that was removed from the list.
 *
 * @param index the index of the element to be removed
 * @return the element previously at the specified position
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

`remove(int)`方法可以用来取出并删除指定位置下的一个节点，同时所有处于其后方的节点都将向前移动一个位置。

```java
/**
 * Removes the first occurrence of the specified element from this list,
 * if it is present.  If this list does not contain the element, it is
 * unchanged.  More formally, removes the element with the lowest index
 * {@code i} such that
 * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
 * (if such an element exists).  Returns {@code true} if this list
 * contained the specified element (or equivalently, if this list
 * changed as a result of the call).
 *
 * @param o element to be removed from this list, if present
 * @return {@code true} if this list contained the specified element
 */
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

/**
 * Removes the first occurrence of the specified element in this
 * list (when traversing the list from head to tail).  If the list
 * does not contain the element, it is unchanged.
 *
 * @param o element to be removed from this list, if present
 * @return {@code true} if the list contained the specified element
 * @since 1.6
 */
public boolean removeFirstOccurrence(Object o) {
    return remove(o);
}

/**
 * Removes the last occurrence of the specified element in this
 * list (when traversing the list from head to tail).  If the list
 * does not contain the element, it is unchanged.
 *
 * @param o element to be removed from this list, if present
 * @return {@code true} if the list contained the specified element
 * @since 1.6
 */
public boolean removeLastOccurrence(Object o) {
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

`remove(Object)`方法和`removeFirstOccurrence(Object)`方法会从头部遍历整个链表，并检查各个节点是否与传入的参数匹配。一旦找到一个匹配的节点就将其删除，并结束操作。`removeLastOccurrence(Object)`方法则是从链表尾部开始查找匹配的节点，并删除第一个匹配到的节点。

## 将`LinkedList`当作栈来操作

上面说过，`LinkedList`可以被当成一个双向队列来操作。那么，如果我们把这个队列的底部“封死”，只操作头部，它是不是就变成了一个栈呢？没错，它是可以这样用的，而且也已经有方法来允许我们这样操作了。实际上，`push`和`pop`的操作，就是直接调用了`addFirst(E)`和`removeFirst()`方法来实现入栈和出栈操作的。

```java
/**
 * Pushes an element onto the stack represented by this list.  In other
 * words, inserts the element at the front of this list.
 *
 * <p>This method is equivalent to {@link #addFirst}.
 *
 * @param e the element to push
 * @since 1.6
 */
public void push(E e) {
    addFirst(e);
}

/**
 * Pops an element from the stack represented by this list.  In other
 * words, removes and returns the first element of this list.
 *
 * <p>This method is equivalent to {@link #removeFirst()}.
 *
 * @return the element at the front of this list (which is the top
 *         of the stack represented by this list)
 * @throws NoSuchElementException if this list is empty
 * @since 1.6
 */
public E pop() {
    return removeFirst();
}
```

## 迭代

`LinkedList`提供了`iterator()`和`listIterator(int)`方法来获取迭代器。实际上这两个方法都将返回一个`ListItr`实例，区别在于`iterator()`是从链表头部开始迭代，而`listIterator(int)`方法则是从指定位置开始迭代。

```java
private class ListItr implements ListIterator<E> {
    // 上次迭代时返回的节点
    private Node<E> lastReturned;

    // 下一次迭代即将返回的节点
    // 其实也是当前指向但仍未取值的节点
    private Node<E> next;

    // 下一个被迭代节点的位置
    private int nextIndex;

    private int expectedModCount = modCount;

    ListItr(int index) {
        // 判断开始迭代的位置是否为链表末尾
        // 如果不是末尾就取出开始迭代位置的节点，否则取出null
        next = (index == size) ? null : node(index);

        // 将nextIndex指向初始迭代位置
        nextIndex = index;
    }

    // 检查有无下一个节点可供迭代
    public boolean hasNext() {
        // 如果下一个迭代位置的下标小于链表长度
        // 就认为还有元素可供迭代
        return nextIndex < size;
    }

    // 获取下一个节点
    public E next() {
        // 检查链表的结构有没有被修改
        checkForComodification();

        // 如果已经没有节点可供迭代
        // 则抛出NoSuchElementException
        if (!hasNext())
            throw new NoSuchElementException();

        // 取出下一个被迭代的节点
        lastReturned = next;

        // next指针像下一个节点移动
        next = next.next;
        nextIndex++;

        // 取出当前被迭代的节点的值
        return lastReturned.item;
    }

    // nextIndex的初始值为0
    // 当它大于0时，就认定该位置的前面仍有节点可供迭代
    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    // 获取上一个节点
    public E previous() {
        // 检查链表的结构有没有被修改
        checkForComodification();

        // 检查前面有无节点可供迭代
        if (!hasPrevious())
            throw new NoSuchElementException();

        // 检查当前节点是否为null，如果是，就说明当前已经处于链表的末尾，那么就返回链表最后一个节点；
        // 如果不是，就返回当前位置的上一个节点
        // 然后设定当前位置和上一次返回位置为上一个节点
        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    // 获取下一个被迭代节点的位置
    public int nextIndex() {
        return nextIndex;
    }

    // 获取上一次被迭代节点的位置
    public int previousIndex() {
        return nextIndex - 1;
    }

    // 删除上一次被迭代的节点
    public void remove() {
        // 检查链表的结构有没有被修改
        checkForComodification();

        // 如果没有上一次被迭代的节点
        // 则抛出IllegalStateException
        if (lastReturned == null)
            throw new IllegalStateException();

        // 取出将被删除节点的下一个节点
        Node<E> lastNext = lastReturned.next;

        // 然后删掉它
        unlink(lastReturned);

        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;

        // 重置上一次被迭代的位置
        lastReturned = null;

        // 因为unlink会使modCound加一
        // 所以这里要同步把expectedModCount加一
        expectedModCount++;
    }

    // 修改上次迭代到的节点的值
    public void set(E e) {
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();

        // 因为lastReturned是某个节点的引用
        // 所以可以直接修改它的值
        lastReturned.item = e;
    }

    // 在下一个被迭代的节点前插入一个节点
    public void add(E e) {
        checkForComodification();
        lastReturned = null;

        if (next == null)
            // 如果已经迭代到链表的末尾，那么就在末尾新增一个节点
            linkLast(e);
        else
            // 否则就在下个被迭代的节点前插入一个节点
            linkBefore(e, next);

        // 游标向后移一位
        nextIndex++;

        // 同步expectedModCount
        expectedModCount++;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (modCount == expectedModCount && nextIndex < size) {
            action.accept(next.item);
            lastReturned = next;
            next = next.next;
            nextIndex++;
        }
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```
