# 配置

### 1、 客户端寻址方式
* 代码中指定NameServer地址  
```java
Producer.setNamesrvAddr(“192.168.8.106:9876”);
```
或
```java
Consumer.setNamesrvAddr(“192.168.8.106:9876”);
```

* Java启动参数中指定NameServer地址  
```shell
-Drocketmq.namesrv.addr=192.168.8.106:9876
```

* 环境变量指定NameServer地址·
```shell
export NAMESRV_ADDR=192.168.8.106:9876
```
* http静态服务器寻址  
客户端启动后，会定时访问一个静态的HTTP服务器，地址如下：  
```
http://jmenv.tbsite.net:8080/rocketmq/msaddr
```
这个URL的返回内容如下：
```
192.168.8.106:9876
```
客户端默认每隔2分钟访问一次这个HTTP服务器，并更新本地的NameServer地址。  
URL已经在代码中写死，可通过修改/etc/hosts文件来改变要访问的服务器，例如在/etc/hosts增加如下配置：
```shell
10.232.22.67   jmenv.taobao.net
```
&nbsp;&nbsp;

### 2、 客户端的公共配置类：ClientConfig

|       参数名        |   默认值    |                            说明                            |
|:------------------:|:-----------:|:----------------------------------------------------------:|
| namesrvAddr        |             | NameServer地址列表，多个nameServer地址用分号隔开              |
| clientIP           | 本机IP      | 客户端本机IP地址，某些机器会发生无法识别客户端IP地址情况，需要应用在代码中强制指定 |
| instanceName       | DEFAULT     | 客户端实例名称，客户端创建的多个Producer，Consumer实际是共用一个内部实例（这个实例包含网络连接，线程资源等） |
| clientCallbackExecutorThreads| 4 | 通信层异步回调线程数        |
| pollNameServerInteval| 30000     | 轮训Name Server 间隔时间，单位毫秒 |
| heartbeatBrokerInterval| 30000   | 向Broker发送心跳间隔时间，单位毫秒  |
| persistConsumerOffsetInterval| 5000   | 持久化Consumer消费进度间隔时间，单位毫秒  |
&nbsp;&nbsp;
&nbsp;&nbsp;

### 3、 Producer配置
|       参数名        |   默认值    |                            说明                            |
|:------------------:|:-----------:|:----------------------------------------------------------:|
| producerGroup      | DEFAULT_PRODUCER| Producer组名，多个Producer如果属于一个应用，发送同样的消息，则应该将它们归为同一组。|
| createTopicKey     | TBW102      | 在发送消息时，自动创建服务器不存在的topic，需要指定key         |
| defaultTopicQueueNums | 4        | 在发送消息时，自动创建服务器不存在的topic，默认创建的队列数     |
| sendMsgTimeout     | 10000       | 发送消息超时时间，单位毫秒                                   |
| compressMsgBodyOverHowmuch | 4096| 消息Body超过多大开始压缩（Consumer收到消息会自动解压缩），单位字节 |
| retryAnotherBrokerWhenNotStoreOK | FALSE| 如果发送消息返回sendResult,但是sendStatus!=SEND_OK,是否重试发送 |
| maxMessageSize     | 131072      | 客户端限制的消息大小，超过报错，同时服务端也会限制（默认128K）  |
| transactionCheckListener |       | 事物消息回查监听器，如果发送事务消息，必须设置                 |
| checkThreadPoolMinSize | 1       | Broker回查Producer事务状态时，线程池大小                      |
| checkThreadPoolMaxSize | 1       | Broker回查Producer事务状态时，线程池大小                     |
| checkRequestHoldMax | 2000       | Broker回查Producer事务状态时，Producer本地缓冲请求队列大小    |
&nbsp;&nbsp;
&nbsp;&nbsp;

### 4、 PushConsumer配置
|       参数名        |     默认值      |                            说明                            |
|:------------------:|:---------------:|:----------------------------------------------------------:|
| consumerGroup      | DEFAULT_CONSUMER| Consumer组名，多个Consumer如果属于一个应用，订阅同样的消息，且消费逻辑一致，则应将它们归为同一组|
| messageModel       | CLUSTERING      | 消息模型，支持以下两种1.集群消费2.广播消费                    |
| consumeFromWhere   | CONSUME_FROM_LAST_OFFSET | Consumer启动后，默认从什么位置开始消费              |
| allocateMessageQueueStrategy | AllocateMessageQueueAveragely | Rebalance算法实现策略              |
| Subscription       | {}              | 订阅关系                                                   |
| messageListener    |                 | 消息监听器                                                 |
| offsetStore        |                 | 消费进度存储                                               |
| consumeThreadMin   | 10              | 消费线程池数量                                             |
| consumeThreadMax   | 20              | 消费线程池数量                                             |
| consumeConcurrentlyMaxSpan | 2000    | 单队列并行消费允许的最大跨度                                |
| pullThresholdForQueue | 1000         | 拉消息本地队列缓存消息最大数                                |
| Pullinterval       | 0               | 拉消息间隔，由于是长轮询，所以为0，但是如果应用了流控，也可以设置大于0的值，单位毫秒 |
| consumeMessageBatchMaxSize | 1       | 批量消费，一次消费多少条消息                                |
| pullBatchSize      | 32              | 批量拉消息，一次最多拉多少条                               |
&nbsp;&nbsp;
&nbsp;&nbsp;

### 5、 PullConsumer配置
|       参数名        |     默认值      |                            说明                            |
|:------------------:|:---------------:|:----------------------------------------------------------:|
| consumerGroup      |                 | Conusmer组名，多个Consumer如果属于一个应用，订阅同样的消息，且消费逻辑一致，则应该将它们归为同一组|
| brokerSuspendMaxTimeMillis | 20000   | 长轮询，Consumer拉消息请求在Broker挂起最长时间，单位毫秒       |
| consumerPullTimeoutMillis | 10000    | 非长轮询，拉消息超时时间，单位毫秒                            |
| messageModel       | BROADCASTING    | 消息模型，支持以下两种：1集群消费 2广播模式                    |
| messageQueueListener |               | 监听队列变化                                                |
| offsetStore        |                 | 消费进度存储                                                |
| registerTopics     |                 | 注册的topic集合                                             |
| allocateMessageQueueStrategy|        | Rebalance算法实现策略                                       |
&nbsp;&nbsp;
&nbsp;&nbsp;

### 6、 Broker配置参数
查看Broker默认配置  
```shell
# mqbroker -m
```
|       参数名        |     默认值      |                            说明                            |
|:------------------:|:---------------:|:----------------------------------------------------------:|
| consumerGroup      |                 | Conusmer组名，多个Consumer如果属于一个应用，订阅同样的消息，且消费逻辑一致，则应该将它们归为同一组|
| listenPort         | 10911           | Broker对外服务的监听端口                                     |
| namesrvAddr        | null            | Name Server地址                                             |
| brokerIP1          | 本机IP          | 本机IP地址，默认系统自动识别，但是某些多网卡机器会存在识别错误的情况，这种情况下可以人工配置。|
| brokerName         | 本机主机名       |                                                            |
| brokerClusterName  | DefaultCluster  | Broker所属哪个集群                                          |
| brokerId           | 0               | BrokerId,必须是大于等于0的整数，0表示Master，>0表示Slave，一个Master可以挂多个Slave，Master和Slave通过BrokerName来配对|
| storePathCommitLog | $HOME/store/commitlog | commitLog存储路径                                     |
| storePathConsumeQueue | $HOME/store/consumequeue | 消费队列存储路径                                 |
| storePathIndex     | $HOME/store/index | 消息索引存储队列                                           |
| deleteWhen         | 4                 | 删除时间时间点，默认凌晨4点                                 |
| fileReservedTime   | 48                | 文件保留时间，默认48小时                                    |
| maxTransferBytesOnMessageInMemory| 262144| 单次pull消息（内存）传输的最大字节数                       |
| maxTransferCountOnMessageInMemory| 32| 单次pull消息（内存）传输的最大条数                             |
| maxTransferBytesOnMessageInDisk| 65535| 单次pull消息（磁盘）传输的最大字节数                          |
| maxTransferCountOnMessageInDisk| 8   | 单次pull消息（磁盘）传输的最大条数                             |
| messageIndexEnable | true            | 是否开启消息索引功能                                          |
| messageIndexSafe   | false           | 是否提供安全的消息索引机制，索引保证不丢                        |
| brokerRole         | ASYNC_MASTER    | Broker的角色 -ASYNC_MASTER异步复制Master  -SYNC_MASTER同步双写Master  -SLAVE |
| flushDiskType      | ASYNC_FLUSH     | 刷盘方式     -ASYNC_FLUSH异步刷盘         -SYNC_FLUSH同步刷盘  |
| cleanFileForciblyEnable | true       | 磁盘满，且无过期文件情况下TRUE表示强制删除文件，优先保证服务可用FALSE标记服务不可用，文件不删除|  
&nbsp;
&nbsp;
&nbsp;

### broker-a.properties  
```java
# 所属集群名字
brokerClusterName=rocketmq-cluster

# broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a

# 0 表示 Master，>0 表示 Slave
brokerId=0

# nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876;rocketmq-nameserver3:9876;

# 在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4

# 是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true

# 是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true

# Broker 对外服务的监听端口
listenPort=10911

# 删除文件时间点，默认凌晨 4点
deleteWhen=04

# 文件保留时间，默认 48 小时
fileReservedTime=120

# commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824

# ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000

# 检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88

# 存储路径
storePathRootDir=/opt/rocketmq/store

# commitLog 存储路径
storePathCommitLog=/opt/rocketmq/store/commitlog

# 消费队列存储路径存储路径
storePathConsumeQueue=/opt/rocketmq/store/consumequeue

# 消息索引存储路径
storePathIndex=/opt/rocketmq/store/index

# checkpoint 文件存储路径
storeCheckpoint=/opt/rocketmq/store/checkpoint

# abort 文件存储路径
abortFile=/opt/rocketmq/store/abort

# 限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000

# Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER

# 刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false

# 发消息线程池数量
#sendMessageThreadPoolNums=128

# 拉消息线程池数量
#pullMessageThreadPoolNums=128

# 强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.1.7
```
&nbsp;
&nbsp;

### broker-a-s.properties  
```java
# 所属集群名字
brokerClusterName=rocketmq-cluster

# broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a

# 0 表示 Master，>0 表示 Slave
brokerId=1

# nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876;rocketmq-nameserver3:9876;

# 在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4

# 是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true

# 是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true

# Broker 对外服务的监听端口
listenPort=10911

# 删除文件时间点，默认凌晨 4点
deleteWhen=04

# 文件保留时间，默认 48 小时
fileReservedTime=120

# commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824

# ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000

# 检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88

# 存储路径
storePathRootDir=/opt/rocketmq/store

# commitLog 存储路径
storePathCommitLog=/opt/rocketmq/store/commitlog

# 消费队列存储路径存储路径
storePathConsumeQueue=/opt/rocketmq/store/consumequeue

# 消息索引存储路径
storePathIndex=/opt/rocketmq/store/index

# checkpoint 文件存储路径
storeCheckpoint=/opt/rocketmq/store/checkpoint

# abort 文件存储路径
abortFile=/opt/rocketmq/store/abort

# 限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000

# Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE

# 刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false

# 发消息线程池数量
#sendMessageThreadPoolNums=128

# 拉消息线程池数量
#pullMessageThreadPoolNums=128

# 强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.1.149
```