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
&nbsp;    
__1.5. org.apache.rocketmq.namesrv.routeinfo.BrokerHousekeepingService__    
BrokerHouseKeepingService实现 ChannelEventListener接口，可以说是通道在发送异常时的回调方法（Nameserver与Broker的连接通道在关闭、通道发送异常、通道空闲时），在上述数据结构中移除已Down掉的Broker。    
org.apache.rocketmq.remoting.ChannelEventListener
```Java ChannelEventListener 接口定义
    void onChannelConnect(final String remoteAddr, final Channel channel);

    void onChannelClose(final String remoteAddr, final Channel channel);

    void onChannelException(final String remoteAddr, final Channel channel);

    void onChannelIdle(final String remoteAddr, final Channel channel);
```    
&nbsp;    
__1.6. NettyServerConfig nettyServerConfig、RemotingServer remotingServer、ExecutorService remotingExecutor__    
这三个属性与网络通信有关，NameServer与Broker、Product、Consume之间的网络通信，基于Netty。
1）NettyServerConfig 的配置含义    
2）Netty线程模型中EventLoopGroup、EventExecutorGroup之间的区别于作用    
3）在Channel的整个生命周期中，如何保证Channel的读写事件至始至终使用同一个线程处理    
首先我们先过一下NettyServerConfig中的配置属性：    
org.apache.rocketmq.remoting.netty.NettyServerConfig
```Java NettyServerConfig 属性定义
    private int listenPort = 8888;
    private int serverWorkerThreads = 8;
    private int serverCallbackExecutorThreads = 0;
    private int serverSelectorThreads = 3;
    private int serverOnewaySemaphoreValue = 256;
    private int serverAsyncSemaphoreValue = 64;
    private int serverChannelMaxIdleTimeSeconds = 120;

    private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
    private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
    private boolean serverPooledByteBufAllocatorEnable = true;

    /**
     * make make install
     *
     *
     * ../glibc-2.10.1/configure \ --prefix=/usr \ --with-headers=/usr/include \
     * --host=x86_64-linux-gnu \ --build=x86_64-pc-linux-gnu \ --without-gd
     */
    private boolean useEpollNativeSelector = false;
```    
&nbsp;    
__1.6.1. serverWorkerThreads__    
含义：业务线程池的线程个数，RocketMQ按任务类型，每个任务类型会拥有一个专门的线程池，比如发送消息，消费消息，另外再加一个其他（默认的业务线程池），默认业务线程池，采用fixed类型，线程个数就是由serverWorkerThreads。    
线程名称：RemotingExecutorThread_    
作用范围：该参数目前主要用于NameServer的默认业务线程池，处理诸如broker,product,consume与NameServer的所有交互命令。    
org.apache.rocketmq.namesrv.NamesrvController
```Java initialize 方法
    public boolean initialize() {

        this.kvConfigManager.load();

        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        // 创建一个线程容量为serverWorkerThreads的固定长度的线程池，该线程池供DefaultRequestProcessor类实现，该类实现具体的默认的请求命令处理。
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        // 就是将DefaultRequestProcessor与创建的remotingExecutor线程池绑定在一起
        this.registerProcessor();

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);

        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            // Register a listener to reload SslContext
            try {
                fileWatchService = new FileWatchService(
                    new String[] {
                        TlsSystemConfig.tlsServerCertPath,
                        TlsSystemConfig.tlsServerKeyPath,
                        TlsSystemConfig.tlsServerTrustCertPath
                    },
                    new FileWatchService.Listener() {
                        boolean certChanged, keyChanged = false;
                        @Override
                        public void onChanged(String path) {
                            if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                log.info("The trust certificate changed, reload the ssl context");
                                reloadServerSslContext();
                            }
                            if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                certChanged = true;
                            }
                            if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                keyChanged = true;
                            }
                            if (certChanged && keyChanged) {
                                log.info("The certificate and private key changed, reload the ssl context");
                                certChanged = keyChanged = false;
                                reloadServerSslContext();
                            }
                        }
                        private void reloadServerSslContext() {
                            ((NettyRemotingServer) remotingServer).loadSslContext();
                        }
                    });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }

        return true;
    }

    private void registerProcessor() {
        if (namesrvConfig.isClusterTest()) {

            this.remotingServer.registerDefaultProcessor(new ClusterTestRequestProcessor(this, namesrvConfig.getProductEnvName()),
                this.remotingExecutor);
        } else {

            this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
        }
    }
```    
具体的命令调用类：org.apache.rocketmq.remoting.netty.NettyRemotingAbstract   
```Java
    /**
     * Process incoming request command issued by remote peer.
     *
     * @param ctx channel handler context.
     * @param cmd request command.
     */
    public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
        final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
        final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
        final int opaque = cmd.getOpaque();

        if (pair != null) {
            Runnable run = new Runnable() {
                @Override
                public void run() {
                    try {
                        RPCHook rpcHook = NettyRemotingAbstract.this.getRPCHook();
                        if (rpcHook != null) {
                            rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                        }

                        final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
                        if (rpcHook != null) {
                            rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                        }

                        if (!cmd.isOnewayRPC()) {
                            if (response != null) {
                                response.setOpaque(opaque);
                                response.markResponseType();
                                try {
                                    ctx.writeAndFlush(response);
                                } catch (Throwable e) {
                                    log.error("process request over, but response failed", e);
                                    log.error(cmd.toString());
                                    log.error(response.toString());
                                }
                            } else {

                            }
                        }
                    } catch (Throwable e) {
                        log.error("process request exception", e);
                        log.error(cmd.toString());

                        if (!cmd.isOnewayRPC()) {
                            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                RemotingHelper.exceptionSimpleDesc(e));
                            response.setOpaque(opaque);
                            ctx.writeAndFlush(response);
                        }
                    }
                }
            };

            if (pair.getObject1().rejectRequest()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                    "[REJECTREQUEST]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
                return;
            }

            try {
                final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
                pair.getObject2().submit(requestTask);
            } catch (RejectedExecutionException e) {
                if ((System.currentTimeMillis() % 10000) == 0) {
                    log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                        + ", too many requests and system thread pool busy, RejectedExecutionException "
                        + pair.getObject2().toString()
                        + " request code: " + cmd.getCode());
                }

                if (!cmd.isOnewayRPC()) {
                    final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                        "[OVERLOAD]system busy, start flow control for a while");
                    response.setOpaque(opaque);
                    ctx.writeAndFlush(response);
                }
            }
        } else {
            String error = " request type " + cmd.getCode() + " not supported";
            final RemotingCommand response =
                RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
        }
    }
```    
该方法比较简单，该方法其实就是一个具体命令的处理模板（模板方法），具体的命令实现由各个子类实现，该类的主要责任就是将命令封装成一个线程对象，然后丢到线程池去执行。
&nbsp;    
__1.6.2. serverCallbackExecutorThreads__    
含义：业务线程池的线程个数，RocketMQ按任务类型，每个任务类型会拥有一个专门的线程池，比如发送消息，消费消息，另外再加一个其他（默认的业务线程池），默认业务线程池，采用fixed类型，线程个数就是由serverCallbackExecutorThreads 。    
线程名称：NettyServerPublicExecutor_    
作用范围：broker,product,consume处理默认命令的业务线程池大小。    



