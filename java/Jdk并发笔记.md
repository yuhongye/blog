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

#### 3. Executor: 解耦任务的提交和任务的执行

用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。

只有一个方法`void execute(Runnable r);`

#### 4. ExecutorService: Executor的子接口，增加了两个能力：

##### 1. 提供了与”终止“相关的方法

一旦调用 shutdown，就不允许再有新的方法提交了，shutdown分两种类型：

1. `shutdown()` 等待已经提交的任务运行完
2. `shutdownNow()` 已提交未运行的任务不会再运行了，同时会尝试终止已经运行的任务，只是尝试而已不提供保证。它会返回已经提交但是还没运行的任务

两种状态:

1. `isShutdown()` 调用过`shutdown()`就返回`true`
2. `isTerminated`等所有任务都执行完成后才返回`true`

##### 2. 提供了可以返回Future以追踪执行结果的方法

#### 5. AbstractExecutorService: 提供了默认实现

它的`newTaskFor()`生成`RunnableFuture`，子类可以覆盖`newTaskFor()`方法。

#### 6. ThreadPoolExecutor

根据JavaDoc，披露了如下几点内容。

1. 线程池大小相关的三个参数：core size, max size，queue。 当线程池的线程数量小于core size时，当有任务到来会创新新的线程；当线程数超过core size时，新任务会优先放到 queue 中，如果queue也满了，才会去创新新线程，直到达到 max size 的限制
2. 如果达到max size 的限制线程池无法提交任务，需要有拒绝策略，拒绝策略是一个接口，可以自定义拒绝策略。JDK提供了4中实现
   * AbortPolicy: 直接拒绝，抛异常
   * CallerRunsPolicy: 提交任务的线程来运行
   * DiscardPolicy: 丢弃这个任务
   * DiscardOldestPolicy: 丢弃最老的任务，这样就会空出一个位置来防止本任务
3. queue在这里起着很大作用，也需要根据不同的情况来选用不同的queue:
   * Direct handoffs，比如使用`SynchronousQueue`，这种情况下queue不提供任务缓冲，一般要求max size要大
   * Unbounded queue, 比如使用LinkedBlockingQueue，这种情况下max size没有任何作用，线程池的数量只会达到core size
   * Bounded queue，这种情况下对queue size 和 max size 进行取舍，如果queue size 大 则 cpu压力小，吞吐量小；如果queue size小，则cpu 压力大，上线文切换带来的损耗大。
4. 这个类提供了两个`protected` hook方法: `beforeExecute` 和 `afterExecute`，通过实现一个子类可以在任务运行前后执行其他逻辑，比如统计、聚合之类的
5. Keep-alive times，任务繁忙时线程池的数量可能超过 core size，当压力较小时需要把超过的线程释放掉，这个参数就是用来控制这部分线程允许的最大空闲时间。`allowCoreThreadTimeOut(boolean)`可以控制是否要回收core size部分的线程
6. 预启动。根据之前的描述，只有当新任务来临时才会触发线程创建，这个类提供了两个方法可以预先创建core size 个线程： `prestartCoreThread()`和`preStartAllCoreThreads`。在创建线程池时提供的queue已经包含了若干任务，在这种情况下就有必要预先启动线程来消耗已存在的任务。





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
  // 虽然signal只会唤醒一个等待的线程，但是这个时候可能有别的未阻塞线程来put
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

1. 如果桶是空的，尝试使用CAS把节点插入到队头，成功则返回，90%的情况下桶里的元素应该应该是0或者1
2. 如果桶不空了，那就锁住队头节点，链表采用尾插法，红黑树正常插入

实现比较精妙的部分是rehash，在插入的时候如果处在rehash的状态，当前线程不是阻塞，而是去帮助rehash操作，大致思路是每个线程分管一部分桶，这样就把rehash也给并行了。

读操作应该是完全并发的，即时在rehash的情况下也不阻塞它。ConcurrentHashMap的桶数是2的次幂，每次扩大一倍，这意味着一个节点在扩容的时候，它要么还在原来的slot上，要么在oldcapacity+slot位置上。据doug lea的注释中说，大概有1/6的entry需要重新构建，其他的可以复用。

为什么TreeBin的搜索比普通的要慢2倍？



wCP9@SyE