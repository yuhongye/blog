# java并发基础知识

#### 0. 教训

1. 应该调用带超时参数的方法，这样能给你一个机会去检查状态；自己写的并发结构也应该提供带超时参数的版本

   

#### 1. Thread

##### join

`Thread.join()`底层使用`wait()`方法实现，当Thread终止时会调用`notifyAll()`方法唤醒所有等待的线程。

##### sleep

`static sleep(millis)`方法使当前线程休眠指定的时间，在此期间，对于已经获得monitors不会丢失，其他尝试获取这些monitors的线程必须等待，这个有点坑

##### Thread.State 它定义的Runnable状态实际上包含了Running和Ready两种状态, 可以通过MangementFactory.getThreadMxBean来获取Thread Dump



#### 2. ThreadLocal

每个线程都有一个该变量的值，一般情况下它应该是一个类的static字段。用两句话来概括ThreadLocal的实现要点：

1. 每个`Thread`类有一个`ThreadLocalMap`对象，它里面存储的是这个Thread的全部ThreadLocal值
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

#### Lock

Lock会保证`lock()`和`unlock()`之间临界区对共享内存变量的操作对其他线程可见，先看一条happens-before原则:

* 如果线程1写入了volatile变量v，接着线程2读取了v，那么线程1写入v及之前的写操作都对线程2可见；volatile的实现可以从两个角度来理解：一是内存屏障，禁止指令重排序；而是volatile变量的写操作会紧跟一个汇编指令LOCK，它的作用是把缓冲区内所有的数据都刷到内存中，是所有的数据，不仅仅是volatile变量。

Lock的实现中依赖了AQS，它内部涉及到对volatile变量的改变，因此它保证了可见性。