# Broker集群配置方式及优缺点
### 1、 单个 Master
&emsp;&emsp;这种方式风险较大，一旦Broker 重启或者宕机时，会导致整个服务不可用，不建议线上环境使用。  
&nbsp;&nbsp;

### 2、 多 Master 模式
&emsp;&emsp;一个集群无 Slave，全是 Master，例如 2 个 Master 或者 3 个 Master  
&emsp;&emsp;优点：配置简单，单个Master 宕机或重启维护对应用无影响，在磁盘配置为 RAID10 时，即使机器宕机不可恢复情况下，由与 RAID10 磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。  
&emsp;&emsp;缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。  

先启动 NameServer，例如机器 IP 为：172.16.8.106:9876
```shell
# nohup sh bin/mqnamesrv &
```

在机器 A，启动第一个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-a.properties &
```

在机器 B，启动第二个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-b.properties &
```
&nbsp;&nbsp;

### 3、 多 Master 多 Slave 模式，异步复制
&emsp;&emsp;每个 Master 配置一个 Slave，有多对Master-Slave，HA 采用异步复制方式，主备有短暂消息延迟，毫秒级。  
&emsp;&emsp;优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为 Master 宕机后，消费者仍然可以从 Slave 消费，此过程对应用透明。不需要人工干预。性能同多 Master 模式几乎一样。  
&emsp;&emsp;缺点：Master 宕机，磁盘损坏情况，会丢失少量消息。  

先启动 NameServer，例如机器 IP 为：172.16.8.106:9876
```shell
# nohup ./mqnamesrv &
```

在机器 A，启动第一个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-async/broker-a.properties &
```

在机器 B，启动第二个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-async/broker-b.properties &
```

在机器 C，启动第一个 Slave
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-async/broker-a-s.properties &
```

在机器 D，启动第二个 Slave
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-async/broker-b-s.properties &
```
&nbsp;&nbsp;

### 4、 多 Master 多 Slave 模式，同步双写
&emsp;&emsp;每个 Master 配置一个 Slave，有多对Master-Slave，HA 采用同步双写方式，主备都写成功，向应用返回成功。  
&emsp;&emsp;优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高  
&emsp;&emsp;缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT 会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。  

先启动 NameServer，例如机器 IP 为：172.16.8.106:9876;172.16.8.107:9876
```shell
# nohup ./mqnamesrv &
```

在机器 A，启动第一个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-sync/broker-a.properties &
```
或：
```shell
# nohup ./mqbroker -n "172.16.8.106:9876;172.16.8.107:9876" -c ../conf/2m-2s-sync/broker-a.properties &
```

在机器 B，启动第二个 Master
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-sync/broker-b.properties &
```
或：
```shell
# nohup ./mqbroker -n "172.16.8.106:9876;172.16.8.107:9876" -c ../conf/2m-2s-sync/broker-b.properties &
```   

在机器 C，启动第一个 Slave
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-sync/broker-a-s.properties &
```
或：
```shell
# nohup ./mqbroker -n "172.16.8.106:9876;172.16.8.107:9876" -c ../conf/2m-2s-sync/broker-a-s.properties &
``` 

在机器 D，启动第二个 Slave
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-2s-sync/broker-b-s.properties &
```
或：
```shell
# nohup ./mqbroker -n "172.16.8.106:9876;172.16.8.107:9876" -c ../conf/2m-2s-sync/broker-b-s.properties &
``` 

&emsp;&emsp;以上 Broker 与 Slave 配对是通过指定相同的brokerName 参数来配对，Master 的 BrokerId 必须是 0，Slave的BrokerId 必须是大与 0 的数。另外一个 Master 下面可以挂载多个 Slave，同一 Master 下的多个 Slave 通过指定不同的 BrokerId 来区分。
***
&nbsp;&nbsp;
&nbsp;&nbsp;


# RocketMQ 安装

### 1、 安装
下载RocketMQ，在每个节点，解压到指定目录
```shell
# unzip rocketmq-all-4.2.0-bin-release.zip
/usr/share/rocketmq-4.2.0
```

进入bin目录  
    注：RocketMQ需要jdk1.7及以上版本  
&nbsp;&nbsp;

### 2、 启动NameServer
```shell
# nohup ./mqnamesrv &
[2] 17938
# nohup: ignoring input and appending output to `nohup.out'
```
查看nohup.out文件中：  
The Name Server boot success.表示NameServer启动成功  

Jps查看NameServer进程  

### 3、 启动BrokerServer a, BrokerServer b
启动master A
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-a.properties &
[1] 17206
```

启动master B
```shell
# nohup ./mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-b.properties &
[1] 14488
```

### 4、 创建topic
```shell
# sh mqadmin updateTopic
usage: mqadmin updateTopic [-b <arg>] [-c <arg>] [-h] [-n <arg>] [-o <arg>] [-p <arg>] [-r <arg>] [-s <arg>]
       -t <arg> [-u <arg>] [-w <arg>]
 -b,--brokerAddr <arg>       create topic to which broker
 -c,--clusterName <arg>      create topic to which cluster
 -h,--help                   Print help
 -n,--namesrvAddr <arg>      Name server address list, eg: 192.168.0.1:9876;192.168.0.2:9876
 -o,--order <arg>            set topic's order(true|false
 -p,--perm <arg>             set topic's permission(2|4|6), intro[2:R; 4:W; 6:RW]
 -r,--readQueueNums <arg>    set read queue nums
 -s,--hasUnitSub <arg>       has unit sub (true|false
 -t,--topic <arg>            topic name
 -u,--unit <arg>             is unit topic (true|false
 -w,--writeQueueNums <arg>   set write queue nums
```

实例：
```shell
# ./mqadmin updateTopic -n 172.16.8.106:9876 -c DefaultCluster -t TopicTest1
create topic to 172.16.8.107:10911 success.
TopicConfig [topicName=TopicTest1, readQueueNums=8, writeQueueNums=8, perm=RW-, topicFilterType=SINGLE_TAG, topicSysFlag=0, order=false]
```

### 5、 删除topic
```shell
# ./mqadmin deleteTopic -n 172.16.8.106:9876 -c DefaultCluster -t TopicTest1
delete topic [TopicTest1] from cluster [DefaultCluster] success.
delete topic [TopicTest1] from NameServer success.
```

### 6、 查看topic信息
```shell
# ./mqadmin topicList -n 172.16.8.106:9876
BenchmarkTest
TopicTest1
broker-a
DefaultCluster
```

### 7、 查看topic统计信息
```shell
# ./mqadmin topicStatus -n 172.16.8.106:9876 -t TopicTest1
#Broker Name            #QID  #Min Offset      #Max Offset             #Last Updated
broker-a                          0     0                     0                     
broker-a                          1     0                     0                      
broker-a                          2     0                     0                     
broker-a                          3     0                     0                     
broker-a                          4     0                     0                      
broker-a                          5     0                     0                     
broker-a                          6     0                     0                     
broker-a                          7     0                     0
```

### 8、 查看所有消费组group
```shell
# ./mqadmin consumerProgress -n 172.16.8.106:9876
```

### 9、 查看指定消费组下的所有topic数据堆积情况
```shell
# ./mqadmin consumerProgress -n 172.16.8.106:9876 -g ConsumerGroupName
```

### 10、 停止broker
```shell
# ./mqshutdown broker
```

### 11、 停止NameServer
```shell
# ./mqshutdown namesrv
```