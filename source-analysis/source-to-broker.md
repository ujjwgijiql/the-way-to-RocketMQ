## Broker概述与同步消息发送原理与高可用设计及思考    
### 1. Broker概述    
Broker在RocketMQ架构中的角色，就是存储消息，核心任务就是持久化消息，生产者发送消息给Broker,消费者从Broker消费消息。
#### RocketMQ网络部署图
[RocketMQ网络部署图]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/reocketMq.png
![RocketMQ网络部署图]
#### RocketMQ逻辑部署结构
[RocketMQ逻辑部署结构]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/reocketMq.png
![RocketMQ逻辑部署结构]
* Producer Group
    用来表示一个发送消息应用，一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息，Producer Group作用如下：
    1.
