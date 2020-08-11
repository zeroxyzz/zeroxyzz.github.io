---
title: Java同步原语Mutex和Semaphore
tags: Java
---

同步原语又叫synchronization primitives，我们都知道在Java中，多线程更新共享资源时，需要锁去保证共享资源的一致性。锁(Lock)就是synchronization primitives中的一种。synchronization primitives主要是用来实现"同步"。

按照类别划分，synchronization primitives可以分为以下几种:

* semaphores
* mutex
* locks
* binary semaphore
* events
* signals
* condition variables
* monitors

上面的分类出自哥伦比亚大学网站的一篇文章，点击进入[Synchronization primitives](http://www.cs.columbia.edu/~hgs/os/sync.html)。精力有限，本文仅介绍Mutex和Semaphores。

<!--more-->

### Mutex & Lock

我们可以把Mutex和Lock当作是一种东西。互斥在Java中很常见，JDK提供的synchronized关键字，在JVM层面提供了互斥; 以及后来Doug Lea提交的Concurrent utils中的ReentrantLock，都是Mutex的实现。

mutex(lock)通常会提供lock和unlock操作，它最大的特点就是"排他"，当多个线程进入临界区(critical section)的代码时，同一时间只能有一个线程能获得对锁，且只有获得锁的线程才能执行unlock操作，其他线程必须等拥有锁的线程执行了unlock操作，才能获得锁。

在Java中，Mutex是很常见的操作，下面的inc()和inc2()都能保证mutex，相信每个有经验的Java程序员都非常熟悉。

```java
public class MutexDemo {
    private static final Lock lock = new ReentrantLock();
    private static int shareVariable;
    
    public synchronized void inc() {
        shareVariable += 1;
    }
    
    public void inc2() {
        try {
            lock.lock();
            shareVariable += 1;
        } finally {
            lock.unlock();
        }
    }
}
```



### Semaphores

Semaphores中文名叫信号量，用于解决并发系统中多个进程(或线程)对公共资源的访问

> In computer science, a semaphore is a variable or abstract data type used to control access to a common resource by multiple processes in a concurrent system such as a multitasking operating system. 

看完上面的定义，你好像又觉得它跟Lock没什么太大的不同，维基百科还有一段定义。

> A semaphore is simply a variable. This variable is used to solve critical section problems and to achieve process synchronization in the multi processing environment. A trivial semaphore is a plain variable that is changed (for example, incremented or decremented, or toggled) depending on programmer-defined conditions.

semaphore是一个**普通的变量**，它用来解决临界区的问题，和实现多进程(线程)的同步问题。具体的操作是在一定的条件下，通过incremented和decremented这个变量来实现同步。

semaphore和mutex最大的区别是，它允许多个线程同时访问公共的资源。维基百科还有一个例子，是关于自习室的。

> Suppose a library has 10 identical study rooms, to be used by one student at a time. Students must request a room from the front desk if they wish to use a study room. If no rooms are free, students wait at the desk until someone relinquishes a room. When a student has finished using a room, the student must return to the desk and indicate that one room has become free.
>
> 完整的例子:  [https://en.wikipedia.org/wiki/Semaphore_(programming)]( https://en.wikipedia.org/wiki/Semaphore_(programming))

假如一个图书馆有10个自习室，一个自习室同一时间只能有被一个学生使用。如果学生想使用自习室，必须要向前台申请，如果没有可用的自习室，那么学生必须要等其他人学完后，才能使用。

听起来十分像AQS中的Shared Lock，我们写个例子来实现上面的例子。这里把自习室改为2个，一共3个人想要获取自习室。

```java
public class SemaphoreDemo {
    private static final Semaphore semaphore = new Semaphore(2);

    public static void main(String[] args) throws Exception {
        ExecutorService es = Executors.newFixedThreadPool(3);
        es.submit(SemaphoreDemo::execute);
        es.submit(SemaphoreDemo::execute);
      	es.submit(SemaphoreDemo::execute);

        TimeUnit.SECONDS.sleep(3);
        es.shutdown();
    }

    private static void execute(){
        try {
            semaphore.acquire();
            System.out.printf("线程 %s 获取到锁\n",Thread.currentThread().getName());
            TimeUnit.SECONDS.sleep(2);
            System.out.printf("线程 %s 释放锁\n",Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
        }
    }
}

//Output
/**
 * 线程 pool-1-thread-1 获取到锁
 * 线程 pool-1-thread-2 获取到锁
 * 线程 pool-1-thread-1 释放锁
 * 线程 pool-1-thread-2 释放锁
 * 线程 pool-1-thread-3 获取到锁
 * 线程 pool-1-thread-3 释放锁
 */
```

从上面的输出来看，线程2并没有在线程1程获得锁后发生阻塞，因为我们创建Semaphore时，permits value 为2，表示同时能有2个线程获得锁，因此线程3必须等待前两个线程其中一个释放才能获取锁。

熟悉AQS的朋友都知道，share lock本质是通过操作AQS的state实现的，下面是Java中Semaphore类的操作。

```java
final int nonfairTryAcquireShared(int acquires) {
  for (;;) {
    int available = getState();
    int remaining = available - acquires;
    if (remaining < 0 ||
        compareAndSetState(available, remaining))
      return remaining;
  }
}

protected final boolean tryReleaseShared(int releases) {
  for (;;) {
    int current = getState();
    int next = current + releases;
    if (next < current) // overflow
      throw new Error("Maximum permit count exceeded");
    if (compareAndSetState(current, next))
      return true;
  }
}
```



### Binary Semaphores和Mutex

通过上面的介绍，或许你会想到，当semaphores的permits value为1时，不就能达到互斥了吗？确实是这样的，这样也叫**binary semaphore**，可以用来实现互斥。比如下面代码就可以达到类似于互斥的效果。

```java
public class BinarySemaphoreDemo {
    private static final Semaphore semaphore = new Semaphore(1);
    private static int shareVariable;

    public static void main(String[] args) throws Exception {
        ExecutorService es = Executors.newFixedThreadPool(10);
        for (int i=0;i<10;i++) {
            es.submit(BinarySemaphoreDemo::fakeMutex);
        }

        es.awaitTermination(2,TimeUnit.SECONDS);
        System.out.printf("shareVariable = %d\n",shareVariable);
        es.shutdown();
    }

    private static void fakeMutex() {
        try {
            semaphore.acquire();
            shareVariable += 1;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
        }
    }
}

//output 
//shareVariable = 10
```

需要注意的是，上面代码不是真正的互斥，因为JUC下的Semaphore类并没有限制其他线程去release。如果要实现真正的互斥，还需要自己扩展功能。

虽然它能实现互斥，不过一般我们不会这么去做，因为它本来就不是设计为互斥的，当你的程序需要用到互斥时，请使用ReentrantLock。

### 结语

synchronization primitives只是一些概念，不要以为只是在Java中存在，Java中的synchronized关键字和ReentrantLock都是synchronization primitives的一些实现。同样的东西，也会出现在linux，windows等操作系统上。

semaphore和mutex共同的目标都是实现synchronization，只是他们的使用场景不太一样。mutex用于互斥的场景，semaphore更倾向于线程之间的调度。