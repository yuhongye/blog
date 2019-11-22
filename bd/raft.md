之前在部门内部分享过一次Raft，看一看《分布式一致性协议-RAFT.ppt》，这里记录一些杂项。

#### 所有节点持久化存储
1. currentTerm: 节点知道的最新term，也是节点以为自己所处的时间点

2. votedFor: 在选举阶段把票投给了谁

3. Log[]: log entries

#### 所有节点在内存中的状态
1. commitedIndex: 该位置及之前的entry已经安全的复制到了major节点上

2. lastApplied: 最后一个应用到SM的entry的index，commitedIndex及之前的都可以安全的应用到SM中

#### Leader特有的
nextIndex[]: 每个节点下一个需要发送的log index

matchIndex[]: 每个节点在log中已经保存的最大index

nextIndex = matchIndex + 1?

### 1. raft协议零碎要点
1. 节点不会投票给比自己旧的节点

2. 节点在一个term内只投一次票

3. 一旦节点发现不比自己老的节点，并且其term比current term大，则
    - set currentTerm = other.term
    - set to follower

4. 节点如果收到了`AppendEntries RPC`请求，意味着新Leader已经产生，变成folloer 

5. Follower不会向leader发送消息，Follower和Leader之间是单向的