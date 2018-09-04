## Broker概述与同步消息发送原理与高可用设计及思考    
### 1. Broker概述    
Broker在RocketMQ架构中的角色，就是存储消息，核心任务就是持久化消息，生产者发送消息给Broker,消费者从Broker消费消息。
