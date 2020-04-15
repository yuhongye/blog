### 谁触发了region split? split的阈值时多少?

目前看到有RangeSplitRequest，但是没有明确看到谁会发送这个请求，Client应该是不用关心region split。现在隐约的在HeartBeater中看到了可能的region split操作，后续还需要仔细看。

目前只看到了最小阈值，也就是低于这个值不允许split。

### Region Split是怎么操作的？

1. 首先找到对应region有多少条数据，然后找出中间的那条数据作为split key
2. 原地分裂，原region的数据范围变成: [start key, split key)
3. 新生成的region数据范围变成：[split key, end key)
4. 对于新生成的region构造对应的RegionEngine，RegionKVService，并生成region raft group

### split对put, get操作的影响是什么？

目前还没看到完整的代码

### 如何保持负载均衡?

region是原地分裂，那么肯定需要全局的负载均衡，目前也还没看到



