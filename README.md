# RocketMQ组件

### 简介
官方简介：  
RocketMQ是一款分布式、队列模型的消息中间件，具有以下特点：  
* 能够保证严格的消息顺序  
* 提供丰富的消息拉取模式  
* 高效的订阅者水平扩展能力  
* 实时的消息订阅机制  
* 亿级消息堆积能力  
&nbsp;&nbsp;

### RocketMQ网络部署图
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
&nbsp;&nbsp;

### 重要概念
* 生产者（Producer）：消息发送方，将业务系统中产生的消息发送到brokers（brokers可以理解为消息代理，生产者和消费者之间是通过brokers进行消息的通信），rocketmq提供了以下消息发送方式：同步、异步、单向。    
* 生产者组（Producer Group）：相同角色的生产者被归为同一组，比如通常情况下一个服务会部署多个实例，这多个实例就是一个组，生产者分组的作用只体现在消息回查的时候，即如果一个生产者组中的一个生产者实例发送一个事务消息到broker后挂掉了，那么broker会回查此实例所在组的其他实例，从而进行消息的提交或回滚操作。    
* 消费者（Consumer）：消息消费方，从brokers拉取消息。站在用户的角度，有以下两种消费者。    
  * 主动消费者（PullConsumer）：从brokers拉取消息并消费。    
  * 被动消费者（PushConsumer）：内部也是通过pull方式获取消息，只是进行了扩展和封装，并给用户预留了一个回调接口去实现，当消息到底的时候会执行用户自定义的回调接口。    
* 消费者组（Consumer Group）：和生产者组类似。其作用体现在实现消费者的负载均衡和容错，有了消费者组变得异常容易。需要注意的是：同一个消费者组的每个消费者实例订阅的主题必须相同。    
* 主题（Topic）：主题就是消息传递的类型。一个生产者实例可以发送消息到多个主题，多个生产者实例也可以发送消息到同一个主题。同样的，对于消费者端来说，一个消费者组可以订阅多个主题的消息，一个主题的消息也可以被多个消费者组订阅。    
* 消息（Message）：消息就像是你传递信息的信封。每个消息必须指定一个主题，就好比每个信封上都必须写明收件人。    
* 消息队列（Message Queues）：在主题内部，逻辑划分了多个子主题，每个子主题被称为消息队列。这个概念在实现最大并发数、故障切换等功能上有巨大的作用。    
* 标签（Tag）：标签，可以被认为是子主题。通常用于区分同一个主题下的不同作用或者说不同业务的消息。同时也是避免主题定义过多引起性能问题，通常情况下一个生产者组只向一个主题发送消息，其中不同业务的消息通过标签或者说子主题来区分。    
* 消息代理（Broker）：消息代理是RockerMQ中很重要的角色。它接收生产者发送的消息，进行消息存储，为消费者拉取消息服务。它还存储消息消耗相关的元数据，包括消费群体，消费进度偏移和主题/队列信息。    
* 命名服务（Name Server）：命名服务作为路由信息提供程序。生产者/消费者进行主题查找、消息代理查找、读取/写入消息都需要通过命名服务获取路由信息。
消息顺序（Message Order）：当我们使用DefaultMQPushConsumer时，我们可以选择使用“orderly”还是“concurrently”。    
  * orderly：消费消息的有序化意味着消息被生产者按照每个消息队列发送的顺序消费。如果您正在处理全局顺序为强制的场景，请确保您使用的主题只有一个消息队列。注意：如果指定了消费顺序，则消息消费的最大并发性是消费组订阅的消息队列数。    
  * concurrently：当同时消费时，消息消费的最大并发仅限于为每个消费客户端指定的线程池。注意：此模式不再保证消息顺序。    
&nbsp;&nbsp;

## 特性
__1、 nameserver__  
相对来说，nameserver的稳定性非常高。原因有二：  
1 、nameserver互相独立，彼此没有通信关系，单台nameserver挂掉，不影响其他nameserver，即使全部挂掉，也不影响业务系统使用。  
2 、nameserver不会有频繁的读写，所以性能开销非常小，稳定性很高。  
&nbsp;&nbsp;

__2、 broker__  
*与nameserver关系*
>连接

&emsp;&emsp;单个broker和所有nameserver保持长连接  

>心跳

&emsp;&emsp;心跳间隔：每隔30秒（此时间无法更改）向所有nameserver发送心跳，心跳包含了自身的topic配置信息。  
&emsp;&emsp;心跳超时：nameserver每隔10秒钟（此时间无法更改），扫描所有还存活的broker连接，  
若某个连接2分钟内（当前时间与最后更新时间差值超过2分钟，此时间无法更改）没有发送心跳数据，则断开连接。  

>断开

&emsp;&emsp;时机：broker挂掉；心跳超时导致nameserver主动关闭连接  
&emsp;&emsp;动作：一旦连接断开，nameserver会立即感知，更新topc与队列的对应关系，但不会通知生产者和消费者  
&nbsp;&nbsp;

*负载均衡*
>一个topic分布在多个broker上，一个broker可以配置多个topic，它们是多对多的关系。

>如果某个topic消息量很大，应该给它多配置几个队列，并且尽量多分布在不同broker上，减轻某个broker的压力。

> topic消息量都比较均匀的情况下，如果某个broker上的队列越多，则该broker压力越大。  

&nbsp;&nbsp;

*可用性*
&emsp;&emsp;由于消息分布在各个broker上，一旦某个broker宕机，则该broker上的消息读写都会受到影响。  
所以rocketmq提供了master/slave的结构，salve定时从master同步数据，  
如果master宕机，则slave提供消费服务，但是不能写入消息，此过程对应用透明，由rocketmq内部解决。  
&nbsp;&nbsp;

这里有两个关键点：
>一旦某个broker master宕机，生产者和消费者多久才能发现？受限于rocketmq的网络连接机制，  
默认情况下，最多需要30秒，但这个时间可由应用设定参数来缩短时间。这个时间段内，  
发往该broker的消息都是失败的，而且该broker的消息无法消费，因为此时消费者不知道该broker已经挂掉。

>消费者得到master宕机通知后，转向slave消费，但是slave不能保证master的消息100%都同步过来了，  
因此会有少量的消息丢失。但是消息最终不会丢的，一旦master恢复，未同步过去的消息会被消费掉。

&nbsp;&nbsp;

*可靠性*
>所有发往broker的消息，有同步刷盘和异步刷盘机制，总的来说，可靠性非常高

>同步刷盘时，消息写入物理文件才会返回成功，因此非常可靠

>异步刷盘时，只有机器宕机，才会产生消息丢失，broker挂掉可能会发生，但是机器宕机崩溃是很少发生的，除非突然断电

&nbsp;&nbsp;

*消息清理*
>扫描间隔

&emsp;&emsp;默认10秒，由broker配置参数cleanResourceInterval决定

>空间阈值

&emsp;&emsp;物理文件不能无限制的一直存储在磁盘，当磁盘空间达到阈值时，不再接受消息，  
broker打印出日志，消息发送失败，阈值为固定值85%  

>清理时机

&emsp;&emsp;默认每天凌晨4点，由broker配置参数deleteWhen决定；或者磁盘空间达到阈值  

>文件保留时长

&emsp;&emsp;默认72小时，由broker配置参数fileReservedTime决定  
&nbsp;&nbsp;

*读写性能*
>文件内存映射方式操作文件，避免read/write系统调用和实时文件读写，性能非常高

>永远一个文件在写，其他文件在读

>顺序写，随机读

>利用linux的sendfile机制，将消息内容直接输出到sokect管道，避免系统调用

&nbsp;&nbsp;

*系统特性*
>大内存，内存越大性能越高，否则系统swap会成为性能瓶颈

>IO密集

>cpu load高，使用率低，因为cpu占用后，大部分时间在IO WAIT

>磁盘可靠性要求高，为了兼顾安全和性能，采用RAID10阵列

>磁盘读取速度要求快，要求高转速大容量磁盘

&nbsp;&nbsp;

__3、 消费者__  
*与nameserver关系*  
>连接

&emsp;&emsp;单个消费者和一台nameserver保持长连接，定时查询topic配置信息，如果该nameserver挂掉，消费者会自动连接下一个nameserver，直到有可用连接为止，并能自动重连。  

>心跳

与nameserver没有心跳  

>轮询时间

&emsp;&emsp;默认情况下，消费者每隔30秒从nameserver获取所有topic的最新队列情况，这意味着某个broker如果宕机，客户端最多要30秒才能感知。该时间由DefaultMQPushConsumer的pollNameServerInteval参数决定，可手动配置。  

*与broker关系*  
>连接

&emsp;&emsp;单个消费者和该消费者关联的所有broker保持长连接。

>心跳

&emsp;&emsp;默认情况下，消费者每隔30秒向所有broker发送心跳，该时间由DefaultMQPushConsumer的heartbeatBrokerInterval参数决定，可手动配置。broker每隔10秒钟（此时间无法更改），扫描所有还存活的连接，若某个连接2分钟内（当前时间与最后更新时间差值超过2分钟，此时间无法更改）没有发送心跳数据，则关闭连接，并向该消费者分组的所有消费者发出通知，分组内消费者重新分配队列继续消费

>断开

&emsp;&emsp;时机：消费者挂掉；心跳超时导致broker主动关闭连接  
&emsp;&emsp;动作：一旦连接断开，broker会立即感知到，并向该消费者分组的所有消费者发出通知，分组内消费者重新分配队列继续消费  

*负载均衡*  
&emsp;&emsp;集群消费模式下，一个消费者集群多台机器共同消费一个topic的多个队列，一个队列只会被一个消费者消费。如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。  

*消费机制*  
>本地队列

&emsp;&emsp;消费者不间断的从broker拉取消息，消息拉取到本地队列，然后本地消费线程消费本地消息队列，只是一个异步过程，拉取线程不会等待本地消费线程，这种模式实时性非常高。对消费者对本地队列有一个保护，因此本地消息队列不能无限大，否则可能会占用大量内存，本地队列大小由DefaultMQPushConsumer的pullThresholdForQueue属性控制，默认1000，可手动设置。  

>轮询间隔

&emsp;&emsp;消息拉取线程每隔多久拉取一次？间隔时间由DefaultMQPushConsumer的pullInterval属性控制，默认为0，可手动设置。  

>消息消费数量

&emsp;&emsp;监听器每次接受本地队列的消息是多少条？这个参数由DefaultMQPushConsumer的consumeMessageBatchMaxSize属性控制，默认为1，可手动设置。  

*消费进度存储*  
&emsp;&emsp;每隔一段时间将各个队列的消费进度存储到对应的broker上，该时间由DefaultMQPushConsumer的persistConsumerOffsetInterval属性控制，默认为5秒，可手动设置。  
如果一个topic在某broker上有3个队列，一个消费者消费这3个队列，那么该消费者和这个broker有几个连接？  
&emsp;&emsp;一个连接，消费单位与队列相关，消费连接只跟broker相关，事实上，消费者将所有队列的消息拉取任务放到本地的队列，挨个拉取，拉取完毕后，又将拉取任务放到队尾，然后执行下一个拉取任务。  
&nbsp;&nbsp;

__4、 生产者__  
*与nameserver关系*  
>连接

&emsp;&emsp;单个生产者者和一台nameserver保持长连接，定时查询topic配置信息，如果该nameserver挂掉，生产者会自动连接下一个nameserver，直到有可用连接为止，并能自动重连。  

>轮询时间

&emsp;&emsp;默认情况下，生产者每隔30秒从nameserver获取所有topic的最新队列情况，这意味着某个broker如果宕机，生产者最多要30秒才能感知，在此期间，发往该broker的消息发送失败。该时间由DefaultMQProducer的pollNameServerInteval参数决定，可手动配置。

>心跳

&emsp;&emsp;与nameserver没有心跳  
&nbsp;&nbsp;

*与broker关系*  
>连接

&emsp;&emsp;单个生产者和该生产者关联的所有broker保持长连接。  

>心跳

&emsp;&emsp;默认情况下，生产者每隔30秒向所有broker发送心跳，该时间由DefaultMQProducer的heartbeatBrokerInterval参数决定，可手动配置。broker每隔10秒钟（此时间无法更改），扫描所有还存活的连接，若某个连接2分钟内（当前时间与最后更新时间差值超过2分钟，此时间无法更改）没有发送心跳数据，则关闭连接。  

>连接断开

&emsp;&emsp;移除broker上的生产者信息  
&nbsp;&nbsp;

*负载均衡*  
&emsp;&emsp;生产者之间没有关系，每个生产者向队列轮流发送消息