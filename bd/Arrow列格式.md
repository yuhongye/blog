列格式的特点：

* 顺序访问的数据相邻性(扫描)
* 支持随机访问，O(1)
* SIMD和向量处理友好
* Relocatable without "pointer swizzling", allowing for true zero-copy access in shared memory

#### 术语

* __Array__ or __Vector__, 数组
* __Slot__: 数组中的一个位置
* __Buffer__ Or __Contiguous memory region__, 指定长度的一片连续空间，空间中的每个byte都可以通过位移访问
* __Physical Layout__: 数组的底层存储，不需要考虑具体的类型
* __Logical type__: 逻辑类型，面向应用提供的类型，底层由physical layout实现，比如Decimal 使用定长的binary实现等



## ValueVector源码分析

#### ValueVector

Notice: 在读写vector前要先分配内存。vector 有几条规则：

* values 需要按顺序写入
* null vector 在写入之前所有的值都是 null
* 对于边长类型，在写入之前 offset vector 里面都是零值
* 在读取 vector之前要先调用 setValueCount
* 当 vector 被读取之后就不能再写入了。

vector 的使用顺序应该是这样的: allocate --> mutate --> setvaluecount --> access --> clear



#### BaseIntVector: 整数类型的 vector 接口，提供了 set/get long 的方法

