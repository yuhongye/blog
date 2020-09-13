# java并发基础知识

#### 0. 教训

1. 应该调用带超时参数的方法，这样能给你一个机会去检查状态；自己写的并发结构也应该提供带超时参数的版本

   

#### 1. Thread

##### join

`Thread.join()`底层使用`wait()`方法实现，当Thread终止时会调用`notifyAll()`方法唤醒所有等待的线程。

##### sleep

`static sleep(millis)`方法使当前线程休眠指定的时间，在此期间，对于已经获得monitors不会丢失，其他尝试获取这些monitors的线程必须等待，这个有点坑

##### interrupt: 状态如何设置，什么情况下会收到异常，状态又是否被清理？

##### Thread.State 它定义的Runnable状态实际上包含了Running和Ready两种状态, 可以通过MangementFactory.getThreadMxBean来获取Thread Dump



#### 2. ThreadLocal

每个线程都有一个该变量的值，一般情况下它应该是一个类的static字段。用两句话来概括ThreadLocal的实现要点：

1. 每个`Thread`类有一个`ThreadLocalMap`对象，它里面存储的是这个Thread的全部ThreadLocal值, map中的key就是每个ThreadLocal对象,value是要set的value
2. 每次`get` 先调用`Thread.getThreadLocalMap`，然后在`ThreadLocalMap`中在获取当前ThreadLocal变量的值

因此在实现ThreadLocalMap时都不需要同步，因为它只会被一个线程操作。值得注意的一点是：ThreadLocalMap中的Entry是weak reference，当线程dead之后，会被清理掉，也就是ThreadLocal的值不用显式的remove。

# Concurrent Collections

#### ArrayBlockingQueue

底层使用数组实现的循环有界队列，它的实现要点是使用一把锁生成两个condition，一个是notEmptyCondition, 一个是notFullCondition：

* put操作时，如果queue is full，await until other get thread signal；一旦put完成，signal other get thread
* get操作时，if queue is empty, await until other put thread signal; 一旦get完成，signal other put thread

```java
// put操作的例程，get操作类似
lock.lock();
try {
  // 注意这里要使用while，因为被唤醒后，可能会被别的线程先拿到锁往队列里放了value，导致队列又满了
  while (isFull()) {
    notFull.await();
  }
  // 入队操作
  enqueue(e);
  notEmpty.signal
} finally {
	lock.unlock();
}
```

这个类的问题在于锁争用比较高，put和get都使用同一把锁，一个可能的优化方向是使用两把锁来减少锁争用，但是两把锁之间的condition如何唤醒是一个问题。

#### LinkedBlockingQueue：吞吐量略高

实现要点：使用了put lock和take lock来减少锁争用，解决了上面ArrayBlockingQueue中的问题.

##### 构造方法中的可见性问题，在单线程中入队，为什么有可见性问题?

```java
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
               	last = last.next = new Node<E>(e);
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

##### 使用两把锁互相唤醒的要点：两把锁分开获取

```java
public void put(E e) {
  Node<E> node = new Node<>(e);
  putLock.lockInterruptibly();
  try {
    // 必须在while循环中，可能被唤醒时，别的线程又put了导致队列满
    while (count.get() == capacity) {
      notFull.await();
    }
    last = last.next = node;
    c = count.getAndIncrement();
   	/**
   	 * 如果此时c==0,那么可能之前有take thread被hold住了，而我们又插入了一个，因此可以唤醒
   	 * 如果c > 0，说明在我们插入之前queue is not empty，在我们插入的时候没有take thread被hold
   	 */
    
    // put唤醒put，因为这个时候还持有锁，所以c不可能增加，如果if条件成立那么就可以唤醒线程
    if (c+1 < capacity) {
      notFull.signal();
    }
  } finally {
    putLock.unLock();
  }
  if (c == 0) {
    takeLock.lock();
    try {
      notEmpty.signal();
    } finally {
      takeLock.unlock();
    }
  }
}
```



### jdk concurrent.locks

#### AQS

参考这篇文章: https://www.cnblogs.com/waterystone/p/4920797.html

#### Lock

Lock会保证`lock()`和`unlock()`之间临界区对共享内存变量的操作对其他线程可见，先看一条happens-before原则:

* 如果线程1写入了volatile变量v，接着线程2读取了v，那么线程1写入v及之前的写操作都对线程2可见；volatile的实现可以从两个角度来理解：一是内存屏障，禁止指令重排序；而是volatile变量的写操作会紧跟一个汇编指令LOCK，它的作用是把缓冲区内所有的数据都刷到内存中，是所有的数据，不仅仅是volatile变量。

Lock的实现中依赖了AQS，它内部涉及到对volatile变量的改变，因此它保证了可见性。



#### Semaphore

底层使用AQS，支持公平和非公平的获取。__在release()时不要求之前调用过acquire()，也就是release是任意的__

```java
Semaphore semaphore = new Semaphore(1);
semaphore.release(9); // now semaphore has 10 permits
```



#### ConcurrentHashMap

##### Java7版本中的ConcurrentHashMap

主要思想：分段锁来分离写竞争，整个ConcurrentHashMap被分成2部分：Segments，每个Segments是一个HashMap，这样每次写操作只需要锁住一个Segments。

更主要的优化：读不加锁，利用了不变性。下面是它的entry

```java
static final class HashEntry<K,V> { 
       final K key;                       // 声明 key 为 final 型
       final int hash;                   // 声明 hash 值为 final 型 
       volatile V value;                 // 声明 value 为 volatile 型
       final HashEntry<K,V> next;      // 声明 next 为 final 型 
}
```

其中需要注意的是`next`是final的，这意味着不能在链表尾部插入，而且一旦插入后，此节点及之后的节点都不会再改变了。当执行remove操作时，找到entry的位置，此entry之前的所有节点重新构造一遍，生成一个新的链表。

##### Java8中的ConcurrentHashMap

Java 7中的实现存在一个问题：限制了最大并发数就是Segments的个数，而且整个结构也不够简洁。在Java8的实现中有两个目标：

1. 保证并发读取的同时降低写入操作的竞争
2. 尽量减小空间占用，至少向HashMap看齐

在不考虑扩容的情况下，ConcurrentHashMap的细粒度锁应该在桶级别，而不是Table或者Segment级别。如果两个写入操作在两个桶中，那它们完全可以并行去做，只有在同一个桶的时候才需要加锁，而且是加载桶中第一个节点上面，因此插入操作可以总结如下：

1. 如果桶是空的，尝试使用CAS把节点插入到队头，成功则返回，正常情况下桶里的元素应该应该是0或者1
2. 如果桶不空了，那就锁住队头节点，链表采用尾插法，红黑树正常插入

实现比较精妙的部分是rehash，在插入的时候如果处在rehash的状态，当前线程不是阻塞，而是去帮助rehash操作，大致思路是每个线程分管一部分桶，这样就把rehash也给并行了。

读操作应该是完全并发的，即时在rehash的情况下也不阻塞它。ConcurrentHashMap的桶数是2的次幂，每次扩大一倍，这意味着一个节点在扩容的时候，它要么还在原来的slot上，要么在oldcapacity+slot位置上。据doug lea的注释中说，大概有1/6的entry需要重新构建，其他的可以复用。