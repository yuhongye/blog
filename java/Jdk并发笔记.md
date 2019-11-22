### 1. ThreadLocal

每个线程都有一个该变量的值，一般情况下它应该是一个类的static字段。用两句话来概括ThreadLocal的实现要点：

1. 每个`Thread`类有一个`ThreadLocalMap`对象，它里面存储的是这个Thread的全部ThreadLocal值
2. 每次`get` 先调用`Thread.getThreadLocalMap`，然后在`ThreadLocalMap`中在获取当前ThreadLocal变量的值

因此在实现ThreadLocalMap时都不需要同步，因为它只会被一个线程操作。值得注意的一点是：ThreadLocalMap中的Entry是weak reference，当线程dead之后，会被清理掉，也就是ThreadLocal的值不用显式的remove。