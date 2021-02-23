#### 0 概述

在内存的实现中有两个主要的组件，一个是JVM级别的memory manager，一个是task级别的task memory manager

###### 管理整个Spark内存的MemoryManager, JVM级别

* `MemoryManger`管理Spark使用的全部JVM内存，它主要管理execution memory和storage memory之间的划分，目前只有一个实现`UnifiedMemoryManager`，execution memory和storage memory可以动态的增减，相互占用
* `MemoryPool`用来记录`MemoryManager`中的execution和storage的memory使用情况

###### 管理单个Task的TaskMemoryManager

* `TaskMemoryManager`管理每个task申请的内存。Task只会和`TaskMemoryManager`交互，绝不直接和JVM级别的`MemoryManager`交互
* `MemoryConsumer` 表示任务中一个单独的操作和数据结构，它是`TaskMemoryManager`的client，它向`TaskMemoryManager`发起内存分配的请求，__当内存紧张时`TaskMemoryManager`会回调`MemoryConsumer`的方法让它把数据spill到磁盘来腾出内存空间__。

![内存结构概览](/Users/caoxiaoyong/Documents/document/blog/bd/images/Memory概览.png)

#### 1. 内存表示

##### MemoryLocation: 表示内存地址， obj + offset

1. 如果是on heap，obj是基址，offset是在obj内的偏移量
2. 如果是off heap，obj is null， offset是虚拟内存地址

##### MemoryBlock extends MemoryLocation，表示一个内存块，额外提供了两个属性：

1. size: 这块内存的大小
2. pageNumber: 主要给`TaskMemoryManager`使用，是一个public字段，`TaskMemoryManager`可以直接修改

#### 2. 内存分配器，提供两个接口:

1. MemoryBlock allocate(size): 分配一段内存
2. free(MemoryBlock): 释放内存

有两个实现：

##### HeapMemoryAlloctor

在分配内存的时候以`long[]`的形式作为MemoryBlock的obj。值得注意的是它的内部对于超过1MB的内存块进行了缓存，缓存的时候把这个`long[]`作为`SoftReference`来避免OOM。但是它的缓存有问题，使用具体的值进行缓存而不是分区间，比如我申请了一个大小为1M+1的内存空间，然后释放，这个时候allocator会把它缓存起来，它的key=1M+1。紧接着我申请了一个1M的空间则不会命中缓存。

##### UnsafeMemoryAllocator 

使用Unsafe直接在堆外申请内存，没什么可说的。

#### 3. MemoryPool: 不要被名字欺骗了，只是记录了pool size, pool used, pool free，pool的大小可以动态

问题：如果它只是记录，为啥还有两个子类？

##### StorageMemoryPool: _memoryStore, memoryMode

需要关注的是`acquireMemory(...): Boolean`，返回值表示是否有足够的空闲内存，如果申请的内存超过了空闲内存会调用`_memoryStore`来驱逐`block`，如果驱逐后空闲内存仍然不够则返回false，申请失败。

##### ExecutionMemoryPool: 它在多个任务间共享，希望任务均分执行内存

在一个executor中可以有N个task，每个task可以分配到的内存在[1/2N, 1/N]，因此它的内部使用`memoryForTask: HashMap[Long, Long]`来记录每隔`taskAttempId`的内存使用量，注意task数是在动态变化的。如果一个task申请的内存还没到1/2N，它不会spill，而是调用`wait()`方法等待别的任务释放内存。

在它的`acquireMemory(): Long`方法中每次都会尝试驱逐storage memory，如果内存不够则会调用`lock.wait()`等待内存，但是它没有调用`lock.notifyAll()`的地方。

问题：如果一个task申请的内存超过了1/N，它在什么时候执行内存释放?

#### 4. MemoryManger: 关注执行内存和存储内存之间共享的问题

MemoryManager抽象类有4个内存池:

* on heap storage memory pool
* on heap execution memory pool
* off heap storage memory pool, 真正发挥作用需要用户配置了off heap
* off heap execution memory pool，真正发挥作用需要用户配置了off heap

一旦off heap不为0，则tungsten memery mode = off heap。

##### UnifiedMemoryManager: 统一的内存管理，如下图所示

![内存结构](/Users/caoxiaoyong/Documents/document/blog/bd/images/spark memory struct.png)

整个类就实现了上图中的逻辑，就是操作execution pool和storage pool的增减，但是它只会操作on heap或者off heap的pool。

#### 5. MemoryConsumer: 只支持tungsten memory

![MemoryConsumer的结构](/Users/caoxiaoyong/Documents/document/blog/bd/images/MemoryConsumer.png)

#### 3. TaskMemoryManager：管理单个任务的内存分配，有一个属性：taskAttempId

问题：管理storage memory吗？看起来全是execution memory的申请。

这个类的复杂性在在于如何把地址编码到64-bit longs中。在off heap下这个比较简单，直接使用虚拟内存地址就行了，但是在on heap模式下内存的表示为`base + offset`，如何把它们编码到long中？计算使用128-bit来保存也是不行的，因为Java对象在内存中是不稳定的，随时会被GC移动位置，为了解决这种情况使用了页表。

1. 在off heap模式下，直接使用虚拟内存地址
2. 在on heap模式下，高13位表示page number，低51位表示页内偏移量，通过page number 去page table array中定位页，然后使用offset在页中定位数据。这样的话，最多有8192个页，每个页可以表示的最大大小受限于long[]的大小，整体可以表示140TB内存。其实单个页内最大为：(2 ^31 - 1) * 8，页内偏移不需要51位。为了实现这个功能有几个属性：
   - MemoryMode tungstenMemoryMode = off heap or on heap;
   - MemoryBlock[] pageTable = new MemoryBlock[PAGE_TABLE_SIZE]；默认就初始化8192个，如果是off heap则每个entry都是null，这个地方存疑。
   - BitSet allocatedPages = new BitSet(PAGE_TABLE_SIZE), 用来跟踪page table 中的entry是否被使用了

