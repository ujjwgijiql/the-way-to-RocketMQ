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
