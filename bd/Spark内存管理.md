## 1. Spark 1.6及之后的内存模型
```java
+---------------+
|               | 缓存数据
|    Storage    | spark.memory.storageFraction
|  ^            | Storage不可以驱逐Execution的内存
+--|------------+ spark.memory.fraction
|  V            | Execution可以驱动Storage，最多驱逐至Storage只剩R
|   Execution   | shuffle, agg
|               |
+---------------+
|   Other       | 用户自定义数据结构，spark元数据，防止OOM
+---------------+
|Reserved(300MB)| 预留数据, 防止OOM
+---------------+
``
Storage和Execution共享一块统一的内存空间，并且可以互相扩展。假设java heap的总大小是H:

* Storage和Execution共享: M = (H-300) * spark.memory.fraction
    - Storage: S = M * spark.memory.storageFraction
    - Execution: E = M - S
* Other: (H-300) - M

## 2. Spark Shuffle
Spark Shuffle同Hadoop Shuffle一样都使用`poll`模式： map task write data to local disk; reduce task通过网络fetch map端的数据。map端的任务就变成：把相同reduce的数据放到一起，方便reduce端读取。

#### 1. HashShuffle
假设有M个Map task, 有R个Reduce task，那么在Map端，会为每一个Reduce生成一个文件；在Reduce端，去每个Map上拉取指定的文件，如果需要combine，则使用HashMap进行合并操作。

缺点：容易OOM，每个Map端打开: M * R个临时文件，性能低。当前版本中已经删除了HashShuffle

#### 2. SortedShuffle
将所有partition的数据都写在一个data文件中，同时用一个Index文件保存每个partition的大小和在data中的偏移量。为什么叫Sorted? 在Map端写文件时会按照parition_id, key进行排序, __Note: Reduce端并不会应用map端的已排序结果，如果需要排序，reduce端会再排一次__

在内存中使用`AppendOnlyMap`，它支持`combine`操作。如果内存不够时，会发生spill操作，在spill之前会使用TimSort进行排序。每次spill都是到单独的文件中，当reduce请求数据时，使用优先队列把这些分散的spill文件合并起来。

缺点：sort会减低性能，有些是不需要排序的。1.2中默认的shuffle。

#### 3. TungstenShuffle

优化手段：

* 将Java对象转成二进制数据，操作直接是在二进制数据上进行的，不需要反序列化
* 缓存感知的排序，对CPU cache友好（思考一个问题：使用对象排序的时候，在比较之前cache中时对象的引用，比较的时候又需要把对象加载到cache中，而对象一般是分散存储的）
* 数据是二进制的，可以直接spill

限制：

1. 不支持aggregation操作，agg操作需要在java object上进行，丧失了再serialized data上操作的优势
2. The shuffle serializer supports relocation of serialized values（kryo和sql serializer支持）
3. 单个record小于128MB

只按照了parition_id进行排序.


1.6中统一了SortedShuffle和TungstenShuffle，自感知的判断使用哪种Shuffle。
