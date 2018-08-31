# RocketMQ源码分析之NameServer(4.3.0)

* NameServer相当于配置中心，维护Broker集群、Broker信息、Broker存活信息、主题与队列信息等。
* NameServer彼此之间不通信，每个NameServer与集群内所有的Broker保持长连接。  

__1. org.apache.rocketmq.namesrv.NamesrvController__

NameServer的核心控制类。

```Java NamesrvController 属性定义
    private final NamesrvConfig namesrvConfig;

    private final NettyServerConfig nettyServerConfig;

    private final ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl(
        "NSScheduledThread"));
    private final KVConfigManager kvConfigManager;
    private final RouteInfoManager routeInfoManager;

    private RemotingServer remotingServer;

    private BrokerHousekeepingService brokerHousekeepingService;

    private ExecutorService remotingExecutor;

    private Configuration configuration;
    private FileWatchService fileWatchService;
```    
&nbsp;   
__1.1. org.apache.rocketmq.common.namesrv.NamesrvConfig__

主要指定nameserver相关配置的目录属性
```Java NamesrvConfig 属性定义
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    private boolean orderMessageEnable = false;
```
1). kvConfigPath(kvConfig.json)    
2). mqhome/namesrv/namesrv.properties     
3). orderMessageEnable：是否开启顺序消息功能，默认为false     
&nbsp;    
__1.2. ScheduledExecutorService scheduledExecutorService__  
NameServer 定时任务执行线程池，一个线程，默认定时执行两个任务：  
    任务1、每隔10s扫描broker,维护当前存活的Broker信息  
    任务2、每隔10s打印KVConfig信息。    
&nbsp;    
__1.3. org.apache.rocketmq.namesrv.kvconfig.KVConfigManager__    
读取或变更NameServer的配置属性，加载NamesrvConfig中配置的配置文件到内存，此类一个亮点就是使用轻量级的非线程安全容器，再结合读写锁对资源读写进行保护。尽最大程度提高线程的并发度。    
```Java KVConfigManager 属性定义
org.apache.rocketmq.namesrv.kvconfig.KVConfigManager
    private final NamesrvController namesrvController;

    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable =
        new HashMap<String, HashMap<String, String>>();
```    
&nbsp;    
__1.4. org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager__    
NameServer数据的载体，记录Broker,Topic等信息。
```Java RouteInfoManager 属性定义
    // NameServer 与 Broker 空闲时长，默认2分钟，在2分钟内Nameserver没有收到Broker的心跳包，则关闭该连接。
    private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
    
    // 读写锁，用来保护非线程安全容器HashMap
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    // 主题与队列关系，记录一个主题的队列分布在哪些Broker上，每个Broker上存在该主题的队列个数。
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    
    // 所有Broker信息，使用brokerName当key,BrokerData信息描述每一个broker信息。
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    
    // broker集群信息，每个集群包含哪些Broker。
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    
    // 当前存活的Broker,该信息不是实时的，NameServer每10S扫描一次所有的broker，根据心跳包的时间得知broker的状态，
    // 该机制也是导致当一个master Down掉后，消息生产者无法感知，可能继续向Down掉的Master发送消息，导致失败（非高可用）。
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```    
1). brokerAddrTable
```Java BrokerData 属性定义
    // broker所属集群
    private String cluster;
    private String brokerName;
    
    // broker 对应的IP:Port,brokerId=0表示Master,大于0表示Slave。
    private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;

    private final Random random = new Random();
```    

