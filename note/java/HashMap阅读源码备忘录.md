网上关于`HashMap`源码解读的已经很多了，这里只记录源码阅读中一些重要的点，仅供个人阅读。

### 初读重点

1. 使用桶来保存数据，`Node[] table`， `Node`是`HashMap`的内部类。 
2. `new HashMap<>(int initialCapacity...)`仅计算了桶的初始大小，并未初始化桶; `new HashMap()`连桶的初始大小都未计算。所以创建一个`HashMap`代价不大。
3. 桶的大小是2^n，这样的好处是可以把mod运算转成mask，但是这样就只能利用hash值的低n位，而有些hash算法本身有较好的分布，所以`HashMap`中将hash值的高位与低位做了一个异或运算:`hash ^ (hash >>> 16)`。
4. `HashMap`是允许key和value都为null，所以`get(key)`返回`null`并不能说明key是否在map中; 必须使用`containsKey(key)`，判断返回的`Node`是否为空。
