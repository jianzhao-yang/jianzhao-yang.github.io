# rocketmq

## nameServ

### 启动流程

#### 启动类 NamesrvStartup

> 启动类的主要目的就是加载并解析配置，添加业务处理器并启动netty

```java
public static NamesrvController main0(String[] args) {
        /**
         * 将命令行参数解析为配置类 namesrvConfig, nettyServerConfig
         */
        NamesrvController controller = createNamesrvController(args);
        /**
         * 【核心】准备核心配置并启动
         */
        start(controller);

}
public static NamesrvController start(final NamesrvController controller) throws Exception {
   		/**
         * 【核心】初始化
         */
    boolean initResult = controller.initialize();
    // virtual-machine shutdown hook. 添加一个初始化但是并没开始的钩子线程，在关闭的时候会调用controller.shutdown();
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));

    //【核心】底层启动netty实例
    controller.start();
    return controller;
}
```

配置类 NamesrvConfig

```java
public class NamesrvConfig {
    // rocketmq位置
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    // kv配置文件位置
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    // 自身配置
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    private boolean orderMessageEnable = false;
}
```

配置类 

```java
public class NettyServerConfig implements Cloneable {
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
     * ../glibc-2.10.1/configure \ --prefix=/usr \ --with-headers=/usr/include \
     * --host=x86_64-linux-gnu \ --build=x86_64-pc-linux-gnu \ --without-gd
     */
    private boolean useEpollNativeSelector = false;
}
```

#### 处理 NamesrvController

> 核心业务和netty服务器控制器

```java
/**
 * NameServer的主要初始化逻辑：
 * 1 从文件中加载nameserver相关配置到configTable（Hashmap）
 * 2 初始化Netty服务器 remotingServer 
 * 3 请求处理器(rocketmq将所有netty请求包装为NettyRequestProcessor进行处理)
 * 4 心跳机制等线程初始化
 * @return
 */
public boolean initialize() {
    //从文件中加载nameserver相关配置到configTable（Hashmap）
    this.kvConfigManager.load();
    //初始化Netty服务器
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

    //核心！ 请求处理器，是NameServer用来处理网络请求的组件，比如register等事件
    this.registerProcessor();

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            //心跳机制：定时扫描不活跃的broker
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            // 定期打印ky配置
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        // 处理ssl重连
        // ---
    }
    return true;
}
```

业务分发类 DefaultRequestProcessor

> 根据request的code对不同类型的请求进行处理

```java
/**
 * AsyncNettyRequestProcessor是抽象类，但是通过一个asyncProcessRequest方法调用了父接口的processRequest方法，processRequest任然交给子类实现
 */
public class DefaultRequestProcessor extends AsyncNettyRequestProcessor implements NettyRequestProcessor {
    private static InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    protected final NamesrvController namesrvController;

    public DefaultRequestProcessor(NamesrvController namesrvController) {
        this.namesrvController = namesrvController;
    }

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
            //请求处理之注册broker的请求
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    //核心注册逻辑
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
            case RequestCode.GET_KVLIST_BY_NAMESPACE:
                return this.getKVListByNamespace(ctx, request);
            case RequestCode.GET_TOPICS_BY_CLUSTER:
                return this.getTopicsByCluster(ctx, request);
            case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
                return this.getSystemTopicListFromNs(ctx, request);
            case RequestCode.GET_UNIT_TOPIC_LIST:
                return this.getUnitTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
                return this.getHasUnitSubTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
                return this.getHasUnitSubUnUnitTopicList(ctx, request);
            case RequestCode.UPDATE_NAMESRV_CONFIG:
                return this.updateConfig(ctx, request);
            case RequestCode.GET_NAMESRV_CONFIG:
                return this.getConfig(ctx, request);
            default:
                break;
        }
        return null;
    }

    @Override
    public boolean rejectRequest() {
        return false;
    }

    public RemotingCommand putKVConfig(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        final RemotingCommand response = RemotingCommand.createResponseCommand(null);
        final PutKVConfigRequestHeader requestHeader =
            (PutKVConfigRequestHeader) request.decodeCommandCustomHeader(PutKVConfigRequestHeader.class);

        this.namesrvController.getKvConfigManager().putKVConfig(
            requestHeader.getNamespace(),
            requestHeader.getKey(),
            requestHeader.getValue()
        );

        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }
}
```

## producer

### 启动流程

#### DefaultMQProducer #start

```java
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
            // 执行MQClientInstance#start，启动相关组件
            mQClientFactory.start();
    		break;	
    }
    // 发送心跳到broker
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    // 定时扫描异步请求结果
    this.timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                RequestFutureTable.scanExpiredRequest();
            } catch (Throwable e) {
                log.error("scan RequestFutureTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

#### MQClientAPIImpl  #start

```java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                // 这里改了状态
                this.serviceState = ServiceState.START_FAILED;
                // 获取 nameServer 的地址
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // 启动远程服务，这个方法只是装配了netty客户端相关配置
                // 注意：1. 这里是netty客户端，2. 这里并没有创建连接
                this.mQClientAPIImpl.start();
                // 启动定时任务
                this.startScheduledTask();
                // pull服务，仅对consumer启作用
                this.pullMessageService.start();
                // 启动负载均衡服务，仅对consumer启作用
                this.rebalanceService.start();
                // 启用内部的 producer，回到原方法执行，神奇的操作未明白用意
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException(...);
            default:
                break;
        }
    }
}
```

这里共有5个定时任务：

1. 定时获取 `nameServer`的地址`MQClientInstance#start`，一开始会调用`MQClientAPIImpl#fetchNameServerAddr`获取`nameServer`
   ，这里也调用了这个方法
2. 定时更新`topic`，的路由信息，这里会去`nameServer`
   获取路由信息，之后再分析
3. 定时发送心跳信息到`nameServer`
   ，在`DefaultMQProducerImpl#start(boolean)`
   中，我们也提到了向`nameServer`
   发送心跳信息，两处调用的是同一个方法
4. 持久化消费者的消费偏移量，这个仅对消费者`consumer`
   有效，后面分析消费者时再作分析
5. 调整线程池的线程数量，不过追踪到最后，发现这个并没有生效，就不多说了

### 发送消息

#### DefaultMQProducer  #send

- 发送消息流程
  - 从nameServ拉取topic
  - 获取重试次数，同步发送才重试
  - 选择要投递的messageQueue
  - 组装message
  - 投递消息

#### MQClientAPIImpl  #sendMessage

```java
/**
 * 同步异步和单向的消息封装
 */
public SendResult sendMessage(
    final String addr,
    final String brokerName,
    final Message msg,
    final SendMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    RemotingCommand request = null;
    String msgType = msg.getProperty(MessageConst.PROPERTY_MESSAGE_TYPE);
    boolean isReply = msgType != null && msgType.equals(MixAll.REPLY_MESSAGE_FLAG);
    if (isReply) {
        if (sendSmartMsg) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE, requestHeader);
        }
    } else {
        if (sendSmartMsg || msg instanceof MessageBatch) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
        }
    }
    request.setBody(msg.getBody());

    switch (communicationMode) {
        case ONEWAY:
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            final AtomicInteger times = new AtomicInteger();
            long costTimeAsync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeAsync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            long costTimeSync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeSync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
        default:
            assert false;
            break;
    }

    return null;
}
```

```java
    /**
     *
     * @param addr 地址
     * @param brokerName
     * @param msg 消息体
     * @param timeoutMillis 超时时间
     * @param request 请求信息
     * @return
     */
private SendResult sendMessageSync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request
) throws RemotingException, MQBrokerException, InterruptedException {
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
    assert response != null;
    return this.processSendResponse(brokerName, msg, response,addr);
}
```

#### RemotingCommand 请求命令描述

```java
// 请求code，用于区分http请求类型： 如心跳、消息等
private int code;
private LanguageCode language = LanguageCode.JAVA;
private int version = 0;
private int opaque = requestId.getAndIncrement();
private int flag = 0;
private String remark;
private HashMap<String, String> extFields;
// 定制请求头
private transient CommandCustomHeader customHeader;
// 序列化类型： json、rocketmq
private SerializeType serializeTypeCurrentRPC = serializeTypeConfigInThisServer;
// 消息体
private transient byte[] body;
```