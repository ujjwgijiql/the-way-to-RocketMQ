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
# nohup sh mqnamesrv &
[2] 17938
# nohup: ignoring input and appending output to `nohup.out'
```
查看nohup.out文件中：  
The Name Server boot success.表示NameServer启动成功  

Jps查看NameServer进程  

### 3、 启动BrokerServer a, BrokerServer b
启动master A
```shell
# nohup sh mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-a.properties &
[1] 17206
```

启动master B
```shell
# nohup sh mqbroker -n 172.16.8.106:9876 -c ../conf/2m-noslave/broker-b.properties &
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
# sh mqadmin updateTopic -n 172.16.8.106:9876 -c DefaultCluster -t TopicTest1
create topic to 172.16.8.107:10911 success.
TopicConfig [topicName=TopicTest1, readQueueNums=8, writeQueueNums=8, perm=RW-, topicFilterType=SINGLE_TAG, topicSysFlag=0, order=false]
```

### 5、 删除topic
```shell
# sh mqadmin deleteTopic -n 172.16.8.106:9876 -c DefaultCluster -t TopicTest1
delete topic [TopicTest1] from cluster [DefaultCluster] success.
delete topic [TopicTest1] from NameServer success.
```

### 6、 查看topic信息
```shell
# sh mqadmin topicList -n 172.16.8.106:9876
BenchmarkTest
TopicTest1
broker-a
DefaultCluster
```

### 7、 查看topic统计信息
```shell
# sh mqadmin topicStatus -n 172.16.8.106:9876 -t TopicTest1
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
# sh mqadmin consumerProgress -n 172.16.8.106:9876
```

### 9、 查看指定消费组下的所有topic数据堆积情况
```shell
# sh mqadmin consumerProgress -n 172.16.8.106:9876 -g ConsumerGroupName
```
