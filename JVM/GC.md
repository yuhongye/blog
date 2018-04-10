# 1. GC算法 Sun Java5内存白皮书
### 1. 设计选择
Parallel: 使用多个线程同时干一件事； Concurrent: 多个事情同时发生

1. Serial vs Parallel

    - Serial只是用单个CPU进行垃圾收集；
    - Parallel将内存划分成parts，每个线程收集一个part，可以同时使用多个CPU进行工作，效率更快，缺点是：复杂，更多的碎片。

2. Concurrent vs Stop-the-world

    - Stop-the-world会暂停应用程序，这是所有的内存区域都不会再修改，因此收集简单，但是应用程序暂停时间长。
    - Conccurrent时应用程序依然在运行，当然也会有一小段暂停。Conccurrent更复杂，当收集时，application也还在修改内存，这会对垃圾收集的性能有影响，同时也要求更大的heap size（Why?）。

3. Compacting vs Non-Compacting vs Copying
当垃圾收回完成时，内存中存活的对象不是连续的，这就造成了内存碎片:

    * Compacting: 把所有存活的对象移动到一起，得到一个大的连续的内存，在allocate的时候速度快、简单。
    * Non-Compacting: 不进行整理，好处时GC快，缺点是：allocate慢，尤其是找到一块适合的内存来存储对象
    * Copying: 把存活的对象拷贝到另外一块内存中，好处是原来的内存区域可以整个清除，坏处是：额外的拷贝时间和内存浪费

### 2. 性能度量指标

1. Throughput: application运行时间/总时间
2. GC Overhead: Throughput的倒数，应用在GC上的时间/总时间
3. Pausing time: application的暂停时间
4. Frequency of GC
5. Footpoint
6. Promptness: object变成垃圾后到所占内存被回收之间的时间

### 3. 分代收集
将整个存储空间分成不同的代，不同代存储不同age的对象，根据每个代的特点可以选择最合适的算法进行优化。在很多语言编程的applicatin都存在weak generational hypothesis:

1. 很多对象存活时间很短
2. older代很少有对younger对象的引用

young generation的内存较小，并且对象存活时间短，因此young generation的GC更频繁，更快，收集效率更高。GC更关注时间。

old generation拥有大多数内存，并且增长缓慢，因此GC次数少，但是每次GC耗时教长。GC更关注空间效率。

young generation中经过几轮GC后存活的对象最终会被promted到old generation。
```java
            Allocation
                |
                V
+----------------------------------+
|A|...|B|...|C|...|     ....       | young generation
+----------------------------------+
                | promotion
                V
+-------------------------------------------+
|A|B|C|   ...   |       ....                | old generation
+-------------------------------------------+

```
### 4. 引用计数法
缺点是无法解决循环依赖，且性能不高。
### 5. 标记-清除
第一步先标记哪些对象是不可达的；第二步清理不可达对象。

缺点：内存碎片
### 6. 标记-清除-整理
在标记-清除的基础上，再进行内存整理。

好处是：无内存碎片；缺点：进行了一次内存整理

### 7. 复制算法
把内存一分为二，运行时只在其中一半分配; 垃圾回收时，把可达的对象移动到另一半，这一半就可以整个清空。

优点：速度快，无碎片； 缺点： 浪费了一半空间。

不过JVM改进了这个算法，它把整个新生代分成:eden, from, to，假设当前from是可以分配的。在进行分配的时候，从eden和from申请内存；当进行垃圾回收时，把eden和from中存活的对象移动到to中，然后eden和from就可以清空了。这里有一点需要注意：如果to的空间不足以装下存活对象，还有老年代呢。这种方式适合垃圾较多，存活对象较少的情况，有研究指出：新生代90%的对象都是朝生夕死的。

### 8. 分代算法
没有一种算法足够优秀，可以一统天下。因此jvm采用了分代的垃圾收集算法：
```java
+----------+-------------+
|  新生代   |   复制算法   |
+----------+-------------+
|  老年代   | 标记清除/压缩 |
+----------+-------------+
```