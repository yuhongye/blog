#### 问题

1. 为什么在builder中配置了`maximumSize`，当超过指定大小时却没有驱逐?当超过指定大小较多时又驱逐了，执行时机是什么？



#### 基础组件

###### 1. Count-min sketch

1. 每个cell使用4bits存储，max count = 15
2. 每个long可以包含16个cell
3. 使用long[] 来构建一个N * 16的矩阵
4. 每个entry 使用 4 个 cell 来保存，取其中最低的值作为 count
5. 每当更新次数达到 sample size，则所有的count减半

_如何确定位置?_

1. 首先计算hash
2. start = (hash & 3) << 2，接下来确定的4个long中会分别使用它们的第start, start + 1, start + 2, start + 3 cell
3. 使用hash分别和4个seed计算出4个index
4. (Index0, start), (index1, start+1), (index2, start+2), (index3, start+3)来确定4个cell

![count-min](/Users/caoxiaoyong/Documents/blog/java/images/count-min-sketch.png)

###### Buffer

也可以先看看JDK的`Stripe64`，caffeine参考它实现了`StripedBuffer`来减少竞争

![caffeine buffer](/Users/caoxiaoyong/Documents/blog/java/images/caffeine-buffer.png)

