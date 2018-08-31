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
__1.3. KVConfigManager__    
读取或变更NameServer的配置属性，加载NamesrvConfig中配置的配置文件到内存，此类一个亮点就是使用轻量级的非线程安全容器，再结合读写锁对资源读写进行保护。尽最大程度提高线程的并发度。    
```Java KVConfigManager 属性定义
org.apache.rocketmq.namesrv.kvconfig.KVConfigManager
    private final NamesrvController namesrvController;

    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable =
        new HashMap<String, HashMap<String, String>>();
```    
&nbsp;   
