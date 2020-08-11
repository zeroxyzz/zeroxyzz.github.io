---
title: CopyOnWriteArrayList中的setArray的迷雾
tags: Java
---

如果有读过CopyOnWriteArrayList的源代码，可能你会发现它的set方法有点怪异，在方法的最后，即使在oldValue == element的情况下，还是调用了一次setArray(es)。

```java
private transient volatile Object[] array;

public E set(int index, E element) {
  synchronized (lock) {
    Object[] es = getArray();
    E oldValue = elementAt(es, index);

    if (oldValue != element) {
      es = es.clone();
      es[index] = element;
    }
    // Ensure volatile write semantics even when oldvalue == element
    setArray(es);
    return oldValue;
  }
}
```

<!--more-->

网上对这个问题的讨论不少，不管是国外还是国内(其实也是抄国外的~)，其中大多观点都集中在happens-before relationship。起因是JLS(Java Language Specification)中有这样一条定义。我复制一段[JLS的定义](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.5)。

> The *happens-before* relation defines when data races take place.
>
> A set of synchronization edges, *S*, is *sufficient* if it is the minimal set such that the transitive closure of *S* with the program order determines all of the *happens-before* edges in the execution. This set is unique.
>
> It follows from the above definitions that:
>
> - An unlock on a monitor *happens-before* every subsequent lock on that monitor.
> - **A write to a `volatile` field ([§8.3.1.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.4)) *happens-before* every subsequent read of that field.**
> - A call to `start()` on a thread *happens-before* any actions in the started thread.
> - All actions in a thread *happen-before* any other thread successfully returns from a `join()` on that thread.
> - The default initialization of any object *happens-before* any other actions (other than default-writes) of a program.

网上的资料普遍都认为setArray是为了保证上述第二条定义，即对一个volatile field的写入操作 happens-before所有读的操作。听起来好像没毛病，即set操作happens-before所有其他线程的get操作，但真的是这样的吗？

事实上，我觉得这种解释有断章取义的嫌疑。首先，上面的定义确实是存在这么一种关系，不过前文说的很清楚，这种关系的前提是数据发生竞争(The *happens-before* relation defines when data races take place)。

在CopyOnWriteArrayList中，get操作是无锁的，而set操作在不存在数据变更时，实际上和get操作根本不存在什么数据竞争，因此根本就不满足JLS中data races take place这个条件。

当我带着这个疑问翻开jdk12的源码时，发现了个更尴尬的问题

```java
/**
 * Replaces the element at the specified position in this list with the
 * specified element.
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E set(int index, E element) {
  synchronized (lock) {
    Object[] es = getArray();
    E oldValue = elementAt(es, index);

    if (oldValue != element) {
      es = es.clone();
      es[index] = element;
      setArray(es);
    }
    return oldValue;
  }
}
```

上面代码是原封不动copy自openjdk-jdk12u的源码，地址在这里[**CopyOnWriteArrayList.java**](https://github.com/AdoptOpenJDK/openjdk-jdk12u/blob/master/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java)。可以看到，openjdk12已经把这个没用的操作移除了，我感觉happens-before一说是不攻自破了。

这问题是有点转牛角尖，感觉意义不是特别大，大部分普通人都不会通读JLS(不然每个人都是Java专家了)，更过分的是用这题来做面试题，这就太折磨人了。

