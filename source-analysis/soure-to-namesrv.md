# RocketMQ源码分析之NameServer(4.3.0)

* NameServer相当于配置中心，维护Broker集群、Broker信息、Broker存活信息、主题与队列信息等。
* NameServer彼此之间不通信，每个NameServer与集群内所有的Broker保持长连接。

__1. org.apache.rocketmq.namesrv.NamesrvController__

NameServer的核心控制类。
``` NamesrvController 属性定义
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
__1.1. org.apache.rocketmq.common.namesrv.KVConfigManager__

主要指定nameserver相关配置的目录属性
``` KVConfigManager 属性定义
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    private boolean orderMessageEnable = false;
```
1). kvConfigPath(kvConfig.json)
2). orderMessageEnable：是否开启顺序消息功能，默认为false
