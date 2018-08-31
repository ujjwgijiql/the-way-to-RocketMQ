# RocketMQ组件

#### RocketMQ网络部署图
[RocketMQ网络部署图]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/reocketMq.png
![RocketMQ网络部署图]

RocketMQ消息队列集群中的几个角色：
* NameServer：在MQ集群中做的是做命名服务，更新和路由发现 broker服务；
* Broker-Master：broker 消息主机服务器；
* Broker-Slave：broker 消息从机服务器；
* Producer：消息生产者；
* Consumer：消息消费者。

RocketMQ集群的通信如下(一部分)：
* Broker启动后需要完成一次将自己注册至NameServer的操作；随后每隔30s时间定期向NameServer上报Topic路由信息；
* 消息生产者Producer作为客户端发送消息时候，需要根据Msg的Topic从本地缓存的TopicPublishInfoTable获取路由信息。如果没有则更新路由信息会从NameServer上重新拉取；
* 消息生产者Producer根据所获取的路由信息选择一个队列（MessageQueue）进行消息发送；Broker作为消息的接收者接收消息并落盘存储。
