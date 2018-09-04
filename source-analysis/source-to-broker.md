## Broker概述与同步消息发送原理与高可用设计及思考    
### 1. Broker概述    
Broker在RocketMQ架构中的角色，就是存储消息，核心任务就是持久化消息，生产者发送消息给Broker,消费者从Broker消费消息。
#### RocketMQ网络部署图
[RocketMQ网络部署图]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/reocketMq.png
![RocketMQ网络部署图]
#### RocketMQ逻辑部署结构
[RocketMQ逻辑部署结构]: https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/logical_deployment%20.png
![RocketMQ逻辑部署结构]
* Producer Group    
    用来表示一个发送消息应用，一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息，Producer Group作用如下：    
    1. 标识一类Producer    
    2. 可以通过运维工具查询这个发送消息应用下有多个Producer实例    
    3. 发送分布式事务消息时，如果Producer中途意外宕机，Broker会主动回调Producer Group内的任意一台机器来确认事务状态    
* Consumer Group    
    用来表示一个消费消息应用，一个Consumer Group下包含多个Consumer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Consumer对象。一个Consumer Group下的多个Consumer以均摊方式消费消息，如果设置为广播方式，那么这个Consumer Group下的每个实例都会消费全量数据。    
&nbsp;     
### 2. Broker存储设计概要     
[RocketMQ数据存储结构]:https://github.com/zhang-jh/the-way-to-RocketMQ/blob/master/images/data_storage_structure.png
![RocketMQ数据存储结构]
#### 从配置文件的角度来窥探Broker存储设计的关注点：    
org.apache.rocketmq.store.config.MessageStoreConfig
```java 属性设置
    // 设置Broker的存储根目录，默认为 $Broker_Home/store
    //The root directory in which the log data is kept
    @ImportantField
    private String storePathRootDir = System.getProperty("user.home") + File.separator + "store";
    
    // 设置commitlog的存储目录，默认为$Broker_Home/store/commitlog
    //The directory in which the commitlog is kept
    @ImportantField
    private String storePathCommitLog = System.getProperty("user.home") + File.separator + "store"
        + File.separator + "commitlog";
    
    // commitlog文件的大小，默认为1G
    // CommitLog file size,default is 1G
    private int mapedFileSizeCommitLog = 1024 * 1024 * 1024;
    
    // ConsumeQueue file size,default is 30W
    private int mapedFileSizeConsumeQueue = 300000 * ConsumeQueue.CQ_STORE_UNIT_SIZE;
    
    // 是否开启consumeQueueExt,默认为false,就是如果消费端消息消费速度跟不上，是否创建一个扩展的ConsumeQueue文件，如果不开启，应该会阻塞从commitlog文件中获取消息，并且ConsumeQueue,应该是按topic独立的。
    // enable consume queue ext
    private boolean enableConsumeQueueExt = false;
    
    // 扩展consume文件的大小，默认为48M
    // ConsumeQueue extend file size, 48M
    private int mappedFileSizeConsumeQueueExt = 48 * 1024 * 1024;
    
    // Bit count of filter bit map.
    // this will be set by pipe of calculate filter bit map.
    private int bitMapLengthConsumeQueueExt = 64;
    
    // 刷写CommitLog的间隔时间，RocketMQ后台会启动一个线程，将消息刷写到磁盘，这个也就是该线程每次运行后等待的时间，默认为500毫秒。flush操作，调用文件通道的force()方法
    // CommitLog flush interval
    // flush data to disk
    @ImportantField
    private int flushIntervalCommitLog = 500;
    
    // 提交消息到CommitLog对应的文件通道的间隔时间，原理与上面类似；将消息写入到文件通道（调用FileChannel.write方法）得到最新的写指针，默认为200毫秒
    // Only used if TransientStorePool enabled
    // flush data to FileChannel
    @ImportantField
    private int commitIntervalCommitLog = 200;
    
    // 在put message( 将消息按格式封装成msg放入相关队列时实用的锁机制：自旋或ReentrantLock)
    /**
     * introduced since 4.0.x. Determine whether to use mutex reentrantLock when putting message.<br/>
     * By default it is set to false indicating using spin lock when putting message.
     */
    private boolean useReentrantLockWhenPutMessage = false;
    
    // Whether schedule flush,default is real-time
    @ImportantField
    private boolean flushCommitLogTimed = false;
    
    // 刷写到ConsumeQueue的间隔，默认为1s
    // ConsumeQueue flush interval
    private int flushIntervalConsumeQueue = 1000;
    // Resource reclaim interval
    private int cleanResourceInterval = 10000;
    // CommitLog removal interval
    private int deleteCommitLogFilesInterval = 100;
    // ConsumeQueue removal interval
    private int deleteConsumeQueueFilesInterval = 100;
    private int destroyMapedFileIntervalForcibly = 1000 * 120;
    private int redeleteHangedFileInterval = 1000 * 120;
       // When to delete,default is at 4 am
    @ImportantField
    private String deleteWhen = "04";
    private int diskMaxUsedSpaceRatio = 75;
    // The number of hours to keep a log file before deleting it (in hours)
    @ImportantField
    private int fileReservedTime = 72;
    
    // 流量控制参数
    // Flow control for ConsumeQueue
    private int putMsgIndexHightWater = 600000;
    // The maximum size of a single log file,default is 512K
    private int maxMessageSize = 1024 * 1024 * 4;
    // Whether check the CRC32 of the records consumed.
    // This ensures no on-the-wire or on-disk corruption to the messages occurred.
    // This check adds some overhead,so it may be disabled in cases seeking extreme performance.
    private boolean checkCRCOnRecover = true;
    // How many pages are to be flushed when flush CommitLog
    
    // 每次flush commitlog时最小发生变化的页数，如果不足该值，本次不进行刷写操作
    private int flushCommitLogLeastPages = 4;
    
    // 每次commith commitlog时最小发生变化的页数，如果不足该值，本次不进行commit操作
    // How many pages are to be committed when commit data to file
    private int commitCommitLogLeastPages = 4;
    
    // 同上
    // Flush page size when the disk in warming state
    private int flushLeastPagesWhenWarmMapedFile = 1024 / 4 * 16;
    
    // 同上
    // How many pages are to be flushed when flush ConsumeQueue
    private int flushConsumeQueueLeastPages = 2;
    private int flushCommitLogThoroughInterval = 1000 * 10;
    private int commitCommitLogThoroughInterval = 200;
    private int flushConsumeQueueThoroughInterval = 1000 * 60;
```    
本次重点关注上述参数，该参数基本控制了生产者--》Broker ---> 消费者相关机制。
接下来从如下方面去深入其实现：   
1）生产者发送消息    
2）消息协议（格式）    
3）消息存储、检索    
4）消费队列维护    
5）消息消费、重试等机制    








