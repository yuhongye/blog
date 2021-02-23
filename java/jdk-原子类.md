##### Stripe64(子类没什么可说的)

###### 设计思路： 当竞争发生的时候引入多个段来降低竞争

这个类有3个重要的属性：

1. base: 所有的更新都先在base上尝试，如果失败了再去分散竞争
2. Cells: 它里面只有一个`volatile long value`，修改使用`CAS`，同时做了CPU缓存行填充。它的大小是2^n，上限是CPU的核数。当竞争发生时，Thread会去cells上更新，通过Thread的hash值确定slot
3. spinlock，在创建cells及其slot时充当锁，控制cells扩容

当想获取总数的时候需要base + cells，没办法得到原子的一致性快照，比如在累加到slot0的时候，slot1上有更新，则最后获取的是slot1更新后的总和。

![stripe64结构示意图](/Users/caoxiaoyong/Documents/blog/java/images/stripe64.png)