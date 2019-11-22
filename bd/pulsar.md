# question of document
1. producer在使用异步发送消息的时候，如何保证消息最终时发送过去的呢？
按照官方文档描述，它会把消息放到一个blocking queue中，这样可以保证消息的按序发送，是不是在blocking queue中时同步发送的？

2. pulsar消息consume mode有shared，它支持multi group的概念吗？