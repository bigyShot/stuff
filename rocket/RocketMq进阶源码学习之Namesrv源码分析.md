**RocketMq进阶源码学习之Namesrv源码分析**

Rocket中Namesrv的角色是注册中心,类似于Kafka依赖的Zookeeper,但是它比Zookeeper更轻量级,因为作为一个MQ的注册中心,不需要Zookeeper那么复杂强大的所有功能(面试可能会问到,回答到这个肯定加分).它是Rocket所有模块中代码最少逻辑最简单的一个模块,如果有想法学习Rocket的源码的话,可以从namesrv模块开始,Namesrv可以集群部署,但每个节点之间互不通信.它的主要作用是保存所有broker的路由信息,以及Topic的分布信息.

从namesrv的main方法开始看,这里就做了两件事,初始化NamesrvController和启动它

```java
public static NamesrvController main0(String[] args) {

    try {
        NamesrvController controller = createNamesrvController(args);
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

在createNamesrvController中主要是初始化namesrv和netty服务器的配置,是的,namesrv是采用的netty来进行网络通信,初始化netty服务器以后与broker和客户端进行通信.

初始化完配置以后开始启动实例

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {

    if (null == controller) {
        throw new IllegalArgumentException("NamesrvController is null");
    }

    //通过KvConfigManager加载配置文件,做一些线程池的初始化和定时任务的配置
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    //JVM关闭的时候执行的钩子函数,做一些关闭netty和线程池的事,目的是为了及时释放资源
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));

    controller.start();

    return controller;
}
```

initialize方法中主要初始化这么些东西,省略掉了一部分不重要的代码

```java
public boolean initialize() {

    //加载配置文件
    this.kvConfigManager.load();

    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    //初始化netty服务器的线程池
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

    //把工作线程池给netty服务器
    this.registerProcessor();

    //定时扫描不活跃的broker,如果过期时间大于2分钟,就把它从map中剔除
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
    .....
```

namesrv启动完毕了,那接下就是它最重要的部分,它是怎么接收和处理来自broker和客户端的请求以及怎么管理broker,topic等元数据的

首先对于元数据的增删改查都在RouteInfoManager这个类中,主要是这么些键值对属性,每个属性干什么的看名字可以看出.可以看到这里的HashMap都是非并发安全的Map,因此这里加了一个**ReentrantReadWriteLock**来对这些键值对在读写时加锁,确保并发安全.

```java
public class RouteInfoManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
    private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

KvConfigManager,主要作用是管理配置文件,在namesrvController初始化实例时加载配置文件到程序内存中,然后通过putKVConfig和persist方法对新增的配置进行管理并持久化到磁盘文件.

```java
public class KVConfigManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    private final NamesrvController namesrvController;
	//依然通过读写锁来保证并发安全
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable =
        new HashMap<String, HashMap<String, String>>();
```



然后就是请求处理器了**DefaultRrequestProcessor,它继承了AsyncNettyRequestProcessor和实现了NettyRequestProcessor**,主要就是实现namesrv作为一个netty服务器处理各种请求的具体逻辑.
首先有一个处理入口方法,这里识别请求的code来做具体不同的业务处理,比如Broker启动时的注册,获取Topic的路由信息等等.

```java
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.PUT_KV_CONFIG:
            return this.putKVConfig(ctx, request);
        case RequestCode.GET_KV_CONFIG:
            return this.getKVConfig(ctx, request);
        case RequestCode.DELETE_KV_CONFIG:
            return this.deleteKVConfig(ctx, request);
        case RequestCode.QUERY_DATA_VERSION:
            return queryBrokerTopicConfig(ctx, request);
        case RequestCode.REGISTER_BROKER:
            Version brokerVersion = MQVersion.value2Version(request.getVersion());
            if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                return this.registerBrokerWithFilterServer(ctx, request);
            } else {
                return this.registerBroker(ctx, request);
            }
        case RequestCode.UNREGISTER_BROKER:
            return this.unregisterBroker(ctx, request);
        case RequestCode.GET_ROUTEINFO_BY_TOPIC:
            return this.getRouteInfoByTopic(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_INFO:
            return this.getBrokerClusterInfo(ctx, request);
        case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
            return this.wipeWritePermOfBroker(ctx, request);
        case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
            return getAllTopicListFromNameserver(ctx, request);
        case RequestCode.DELETE_TOPIC_IN_NAMESRV:
            return deleteTopicInNamesrv(ctx, request);
.......省略一部分case
    }
    return null;
}
```

Namesrv的主要代码和逻辑就在这里了,总结一下:

1.Namesrv启动的时候会先初始化整体和netty配置(包括初始化类,以及从命令行读取配置)

2.启动实例,初始化一些线程池去做定时任务(比如扫描broker的存亡)

3.以DefaultRequestProcessor作为Netty与外界通信的入口,接收各种各样的请求,然后转发到具体类处理业务

4.通过KvConfigManager和RouteInfoManager管理元数据信息和配置数据

这篇文章只是过一下整体概览,很多具体方法都没有细致分析,其实这些方法里都是一些基础的业务逻辑了,明白了整体的走向流程以及原理就差不多足够了,刚开始学习一个项目源码不必追求太过精细,明白了整体,以后如有需要再回来看细节部分也很简单.