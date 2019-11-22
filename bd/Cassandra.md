### Snitch
snitch有两个主要的功能：

* 它报告网络拓扑信息，这样可以高效的转发请求：是说转发给性能更高的副本节点吗？

* 它提供了datacenter，rack的信息，这样可以让cassandra尽最大努力把副本分散开来，避免把副本存在在一个地方（比如同一个rack)

dynamic snitching monitor read latencies to avoid reading from hosts that have slowed down. 是说它会定期的去跟跟各个节点交互，执行一个相对耗时的计算node score的操作，通过这个score来避免去读slow node，加重集群的不均衡。