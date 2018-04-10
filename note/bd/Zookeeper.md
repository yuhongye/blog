Zookeeper学习过程中的记录。
# 1. 问题

1. Zookeeper提供了什么样的一致性？强一致性还是最终一致性?

# 2. 基础
Zookeeper提供了一小部分调用方法组成的类文件系统api。

每个znode有一个version,它随着每次数据变化而自增，version类似CAS，compareAndGet, compareAndSet等，只有当前version等于client传递的version时，才能操作成功。

会话：client提交给zookeeper的所有操作均关联在一个会话上，当会话失效时，client创建的所有临时znode都会被删除。当client无法与server连接时，会透明的选择其他可联通服务器，仍然是同一个会话，__Note: client不能连接一个比它老的服务器__。会话提供了顺序保证：同一个会话的请求会以FIFO顺序执行。会话的生命周期：
```java 
                       +-------------------------+
                       |                         |
                       |                         V
+------------+   +----------+    +---------+   +------+
|NO_CONNECTED|-->|Connecting|<-->|Connected|-->|CLOSED|
+------------+   +----------+    +---------+   +------+
```
会话过期：server才可以申明会话过期，比如client与server因超时而断开连接，client保持Connecting状态；client因网络分区与server断开连接，client依然保持Connecting状态。client只能从server处知悉会话已经过期，不过它可以主动关闭会话。

会话超时时间t: 在server端如果经过t时间没有收到会话的任何消息，则生命会话过期；在client端，经过t/3时间未收到消息，向server发送心跳，经过2t/3后，client开始寻找其他服务器进行连接。

`zkServer.sh start -foreground`这个命令提供了大量的详细信息输出。
