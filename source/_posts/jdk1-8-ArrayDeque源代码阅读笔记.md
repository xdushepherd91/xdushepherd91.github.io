---
title: jdk1.8--ArrayDeque源代码阅读笔记
date: 2019-09-09 22:26:40
tags:
---

### 类注释总结

ArrayDeque是Deque接口的一个可变数组实现，该实现有以下一些特点：

1. 没有容量限制。
2. 非线程安全。这意味着在多线程中使用的时候，需要添加一定的同步手段。
3. 不接受null数据。
4. 作为一个栈比Stack类要快。
5. 作为队列比LinkedList类要快。

同其他的集合类一样，该类的iterator方法是fail-fast的，除了通过iterator本身的remove方法，任何在iterator创建之后对deque的删除操作，都会出发ConcurrentModificationException异常。

### 类初始化相关

#### 实例变量

可以看到以下几点：
1. ArrayDeque内部使用的是一个对象数组来存储元素的。
2. 使用了一个头指针head，指向了当前可以删除的元素下标。
3. 使用了一个尾指针tail，指向了下一个可以添加数据的位置下表。

```java
    transient Object[] elements;
    transient int head;
    transient int tail;
```

### 私有工具方法介绍

#### calculateSize方法

calculateSize方法通过位运算的方式，得到了一个恰好大于入参numElements的一个2的次幂。方法解释如下：
1. 对于任何入参numElements，必定有一个最高位1，通过一次右移，并与原来的numElements进行或操作。必定得到一个高两位为1的数字。
2. 对1中得到的高两位为1的数字a，再右移两位得到数字b ，两者再进行或操作，得到一个高四位为1的数字
3. 以此类推，我们可以把从入参numElements最高位1后的所有位都变为1。
4. 由于java。中的int位32位，那么我们可以保证在右移16位后一定可以把任何一个int型入参变为高位后所有位都为1的数字
5. 对上述得到的数字加1，即得到了一个2的次幂，且恰好大于入参numElements

```java
private static final int MIN_INITIAL_CAPACITY = 8;

// ******  Array allocation and resizing utilities ******

private static int calculateSize(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    return initialCapacity;
}
```

#### allocateElements方法

参考上一节，allocateElements方法不难理解。

```java
private void allocateElements(int numElements) {
    elements = new Object[calculateSize(numElements)];
}
```
#### doubleCapacity方法

顾名思义，对容量进行翻倍。具体看看如何实现：
1. 这里假定头指针和尾指针相遇了。
2. 计算当前数组的长度n。
3. 计算头指针右侧的数据个数。
4. 对n进行左移一位操作，得到新的数组容量newCapacity。n为2的次幂。
5. 初始化一个容量为newCapacity的新的数组。
6. 将原来的数组元素复制到新数组中。

```java

private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

