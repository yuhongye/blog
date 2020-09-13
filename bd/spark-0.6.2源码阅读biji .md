`DAGScheduler`使用生产者/消费者模型，它定义了run方法，会在init block中启动单线程作为消费者，消费逻辑就是它的run方法。使用`LinkedBlockingQueue`持有event，event的基类是`trait DAGSchedulerEvent`，它没有任何属性，也不包含任何方法，可能的event有如下几类:

* JobSubmitted
* CompletionEvent
* HostLost
* TaskSetFailed
* StopDAGScheduler



RDD的操作通过`DAGScheduler.runJob`来运行，这个方法逻辑：把操作封装成`JobSumited`放到队列中，同时构造一个`JobWaiter`等待任务完成。

