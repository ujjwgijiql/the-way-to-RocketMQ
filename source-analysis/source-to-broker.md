## Broker概述与同步消息发送原理与高可用设计及思考    
### 1. Broker概述    
Broker在RocketMQ架构中的角色，就是存储消息，核心任务就是持久化消息，生产者发送消息给Broker,消费者从Broker消费消息。
#### RocketMQ网络部署图
[RocketMQ网络部署图]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/reocketMq.png
![RocketMQ网络部署图]
#### RocketMQ逻辑部署结构
[RocketMQ逻辑部署结构]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/logical_deployment%20.png
![RocketMQ逻辑部署结构]
* Producer Group    
    用来表示一个发送消息应用，一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息，Producer Group作用如下：    
    1. 标识一类Producer    
    2. 可以通过运维工具查询这个发送消息应用下有多个Producer实例    
    3. 发送分布式事务消息时，如果Producer中途意外宕机，Broker会主动回调Producer Group内的任意一台机器来确认事务状态    
* Consumer Group    
    用来表示一个消费消息应用，一个Consumer Group下包含多个Consumer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Consumer对象。一个Consumer Group下的多个Consumer以均摊方式消费消息，如果设置为广播方式，那么这个Consumer Group下的每个实例都会消费全量数据。    
&nbsp;     
### 2. Broker存储设计概要     
[RocketMQ数据存储结构]:https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/data_storage_structure.png
![RocketMQ数据存储结构]
#### 从配置文件的角度来窥探Broker存储设计的关注点：    


