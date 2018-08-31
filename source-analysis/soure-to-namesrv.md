# RocketMQ源码分析之NameServer

* NameServer相当于配置中心，维护Broker集群、Broker信息、Broker存活信息、主题与队列信息等。
* NameServer彼此之间不通信，每个NameServer与集群内所有的Broker保持长连接。

__1. org.apache.rocketmq.namesrv.NamesrvController__
