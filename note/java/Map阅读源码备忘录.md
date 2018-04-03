JDK提供了几种`Map`实现，这里记录各种实现的一些要点。

# 1. HashMap
网上关于`HashMap`源码解读的已经很多了，这里只记录源码阅读中一些重要的点，仅供个人阅读。

### 初读重点

1. 使用桶来保存数据，`Node[] table`， `Node`是`HashMap`的内部类。 
2. `new HashMap<>(...)`并未初始化桶，所以创建一个`HashMap`代价不大，当第一次插入元素的时初始化桶。
3. 桶的大小是2^n，这样的好处是可以把mod运算转成mask，但是这样就只能利用hash值的低n位，而有些hash算法本身有较好的分布，所以`HashMap`中将hash值的高位与低位做了一个异或运算:`hash ^ (hash >>> 16)`。
4. `HashMap`是允许key和value都为null，所以`get(key)`返回`null`并不能说明key是否在map中; 必须使用`containsKey(key)`，判断返回的`Node`是否为空。
5. 桶中的slot本身保存了一个entry。
6. 插入时如果是链表，则插入到链表的尾部
7. `modCount`： hash map结构变化次数，元素个数的变化，不包括已存在key的value更新
8. `clear()`方法简单的把所有slot都置空，只是一次遍历`table`的耗时

### 必要的知识
`Node`类：链表使用，也是桶的类型
```java
static class Node<K, V> ... {
    /**
     * Notice: 保存的是key处理后的hash值: key.hash ^ (key.hash >>> 16)
     */
    final int hash;
    final K key;
    V value;
    Node<K, V> next;
}
```

`TreeNode`类: 红黑树使用
```java
static class TreeNode<K, V> 间接继承 Node<K, V> {
    TreeNode<K, V> parent;
    TreeNode<K, V> left;
    TreeNode<K, V> right;
    TreeNode<K, V> prev;
    boolean red;
}
```

#### 红黑树的优化
Jdk 8的HashMap引入了红黑树进行优化：当某个slot的链表元素过多时，把链表转成红黑树；当红黑树元素过少时，把红黑树转成链表，有3个参数控制这个行为：
    
    * 链表元素个数 > 8 && 桶元素个数小于64时，resize而不是转红黑树
    * 链表元素个数 > 8时，把链表转成红黑树
    * 红黑树元素小于6时，转成链表

桶中的slot保存的可能是链表的`Node`或者红黑树的`TreeNode`，由于`TreeNode`是`Node`的子类，所以通过判断 `slot instanceof TreeNode`来判断当前保存的是否为红黑树。__顺序遍历__： 如论是链表还是红黑树都可以通过`Node.next`进行顺序遍历。

关于红黑树的代码，等学习完红黑树后再更新在这里。

#### resize过程
resize: 桶的个数扩大一倍, `threshold`变大一倍，同时进行rehash。当slot中entry较少时，使用list保存，它的rehash代码比较巧妙:

__推论__: 桶中slot i中的node，经过rehash后，要么继续在slot i中，要么在slot (oldCap + i)中

__证明__:

1. 桶原来的大小oldCap是2的幂，假设oldCap = 2^k， node确定所属slot的算法：node.hash & (oldCap - 1), 也就是node.hash的最低k位的值;
2. newCap = 2^(k+1), 决定node所属slot变成了node.hash中低(k+1)位的值；现在就分成了两种情况：

        - node.hash的第(k+1)位为0, 也就是(node.hash & oldCap) == 0，那么它会在原来的slot
        - node.hash的第(k+1)位为1, 它的slot就变成了: 2^k(第k+1位为1表示的值) + 低k位的值，也就是: oldCap + i, i是该node在rehash之前所处的slot

```java
    // 原来slot j处的链表
    Node<K,V> loHead = null, loTail = null;
    // oldCap + j slot处的链表
    Node<K,V> hiHead = null, hiTail = null;
    Node<K,V> next;
    do {
        next = e.next;
        // 第k+1位为0， 决定它所处slot的依然是低k位的值，还在原来的slot中
        if ((e.hash & oldCap) == 0) {
            if (loTail == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
        }
        // 说明高位为1, 会在当前slot index + oldCap slot中
        else {
            if (hiTail == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
        }
    } while ((e = next) != null);
    if (loTail != null) {
        loTail.next = null;
        // j: 当前的slot； 注意在rehash的过程中，根据上述推论，可以得知：每个slot只会被赋值一次
        newTab[j] = loHead;
    }
    if (hiTail != null) {
        hiTail.next = null;
        newTab[j + oldCap] = hiHead;
    }
```

##### threshold
`threshold`的作用：当map size > threshold时，resize。`threshold = (table.length or default initial capacity) * loadFactor`，在看代码时有一个迷惑人的地方：`new HashMap(intialCapacity...)`时使用了`threshold`来保存initialCapacity。因此当第一次出入entry时，`table` is null, 首先需要resize:

    * `table = new Node[threshold]`
    * `threshold = table.length * loadFacto`

# 2. LinkedHashMap extends HashMap
在`HashMap`的基础上又维护了一个double-linked list来保持顺序：

* 插入顺序，遍历时得到的是插入顺序
* 访问顺序，每次访问时都会把当前entry放到队列尾部（最新的)

提供了一个方法`protected boolean removeEldestEntry(Map.Entry<K, V> eldes)`，默认返回`false`，子类可以返回`true`来删除最老元素的目的，配合访问顺序，Map可以作为一个LRU Cache。
```java
// 元素个数上限，超过删除最老的
private int maxSize;
/**
 * 删除最老的元素
 */
@Override
protected boolean removeEldestEntry(Map.Entry<K, V> eldst) {
    return size() > maxSize;
}
```

# 3. TreeMap
