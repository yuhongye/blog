通过raft-lib来组成一个raft group，它包含若干个`Node`，每个Node的状态可能是Leader, Follower或者candidate，其中有一个是Leader，raft group之间需要通过rpc来交换raft 状态信息。由于counter只保存一个value值，因此在整个系统中只存在一个raft group。

counter这个raft group提供了分布式原子计数服务，它通过网络向client提供功能，因此我们还需要一个与业务交互的rpc，raft rpc和业务rpc可以共用。

一次完整的走raft log的请求流程是：

1. 通过rpc服务接受到了请求，调用对应的processor来处理请求
2. 业务processor将请求生成Task，调用Node.apply(Task)，想Raft group提交任务
3. Node使用disruptor来支持生产者/消费者模型，它将task放进生产者/消费者队列中，然后需要通过raft算法把这个请求复制到别的node上，当超过半数的node在本地复制了task后，就认为它可以安全的提交了
4. 对于可以提交的task，调用StateMachine来应用状态，对于leader而言，它需要判断task的closure是否为null，closure存在的话需要调用它的done来返回响应：通常是向client返回结果



### 如何开发新的服务

最重要的是开发StateMachine，它实现了业务逻辑。同时要启动server