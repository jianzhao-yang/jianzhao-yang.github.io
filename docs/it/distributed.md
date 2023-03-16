# 分布式系统常见面试题

## SpringCloud

### SpringBoot

1. SpringBoot有哪些优点？
   - 内置servlet容器（tomcat、jetty等）
   - 简化默认配置（约定大于配置）
   - 简化maven依赖（starter）
   - 自动配置 autoconfiguration
   - yml、properties配置文件，dev、prd自动选择配置
   - 自动装配bean
   - 命令行cli
   - 监视actuator 
2. Spring boot常用注解？
   - 配置类：
     - @SpringBootApplication：包含了@ComponentScan、@Configuration和@EnableAutoConfiguration注解
   - bean注入类：
     - @Bean 通过方法返回一个bean；@Component 注册一个bean；@Controller；@Service；@Repository
   - controller层：
     - @RestController；@RequestMapping；@ResponseBody；@PathVariable；@RequestParam
3. 如何使用热启动（修改代码自动重启）？
   - 引入spring-boot-devtools 
4. springboot常用的starter有哪些 ？
   - spring-boot-starter-web 嵌入tomcat和web开发需要servlet与jsp支持 
   - spring-boot-starter-data-jpa 数据库支持 
   - spring-boot-starter-data-redis redis数据库支持 
   - mybatis-spring-boot-starter 第三方的mybatis集成starter
5. springboot自动配置的原理 
   在spring程序main方法中 添加@SpringBootApplication或者@EnableAutoConfiguration 
   会自动去maven中读取每个starter中的spring.factories文件 该文件里配置了所有需要被创建spring容器中的bean
6. 常见的starter会包几个方面的内容？分别是什么？
   - 常见的starter会包括下面四个方面的内容 
   - 自动配置文件，根据classpath是否存在指定的类来决定是否要执行该功能的自动配置。 
   - spring.factories，非常重要，指导Spring Boot找到指定的自动配置文件。 
   - endpoint：可以理解为一个admin，包含对服务的描述、界面、交互(业务信息的查询)。 
   - health indicator:该starter提供的服务的健康指标。
7. 自定义springboot-starter注意事项？
   - springboot默认scan的包名是其main类所在的包名。如果引入的starter包名不一样，需要自己添加scan。 @ComponentScan(basePackages = {"com.xixicat.demo","com.xixicat.sms"}) 
   -  对于starter中有feign的，需要额外指定 @EnableFeignClients(basePackages = {"com.xixicat.sms"})
   -  对于exclude一些autoConfig @EnableAutoConfiguration(exclude ={MetricFilterAutoConfiguration.class})
8. 常见Conditional*注解？
   - @ConditionalOnBean
   - @ConditionalOnClass
   - @ConditionalOnMissingBean
   - @ConditionalOnMissingClass
   - @ConditionalOnWebApplication
   - @ConditionalOnResource
   - @ConditionalOnProperty

### 网关

#### 为什么Spring Cloud GateWay和Zuul同时出现？

​	因为zuul1.x基于servlet是阻塞的，2.x又经常停更跳票，所以spring自己出了gateway代替

#### 微服务网关可以有哪些用途？

- 屏蔽内部细节
- 反向路由
- 安全认证
- 限流熔断
- 日志监控
- 灰度发布

### Ribbon 负载均衡

1. 常用注解：@LoadBalanced

### [eureka](eureka.md) 服务发现 

1. 常用注解：@EnableDiscoveryClient
2. 常用配置： 
3. 

### Feign 服务调用

1. 常用注解： @EnableFeignClients、@FeignClient

### Hystrix 服务熔断

1. 常用注解： @EnableHystrix、@EnableCircuitBreaker、@HystrixCommand(fallbackMethod = “”)、@EnableHystrixDashboard、@CacheResult

### Apollo 配置中心

1. 常用注解： @EnableApolloConfig、@ApolloConfig、@ApolloConfigChangeListener

## JUC

### AbstractQueuedSynchronizer（AQS）

​		https://www.jianshu.com/p/da9d051dcc3d

- 内部维护了一个CLH实现的FIFO队列，用于线程等待获取资源；
- 又维护了一个volatile的变量state，用于代表资源；
- 资源有两种模式：互斥的（一次只能一个线程使用，如ReentrantLock）和共享的（有限个数线程共同使用，如Semaphore/CountDownLatch）；

### Unsafe

- 为什么是unsafe？
  - 核心方法都是native的，底层直接操作引用和内存，会造成不可知的风险；故只有JDK的类能使用（限制了只有BootStrapClassLoader加载的类可以使用）
  - Unsafe类可以使用反射的方式绕过限制直接使用
- ABA问题的解决：AtomicStampedReference和 AtomicMarkableReference；使用版本号/时间戳解决

## MQ

### RabbitMQ

- 核心概念

  ![https://blog.csdn.net/xinbaobaoer/article/details/54907793](images\rabbitmq.png)

  - ConnectionFactory：与RabbitMQ服务器连接的管理器
  - Connection：与RabbitMQ服务器的连接
  - Channel：与Exchange的连接
  - Exchange：接受消息提供者（生产者）的消息，并根据消息的RoutingKey和Exchange绑定的BindingKey分配消息
  - Queue：存储消息接收者（消费者）的消息
  - RoutingKey：指定当前消息被谁接受
  - BindingKey：指定当前Exchange下，什么样的RoutingKey会被下派到当前绑定的Queue中

- 死信队列

  - https://www.cnblogs.com/toov5/p/10288260.html
  - 无法被消费的消息会被归到死信队列中去
  - 可以根据死信队列的原理，设置消息的失效时间，构造延时队列

### RocketMQ

## Zookeeper

### zookeeper是什么?

- zookeeper是一个分布式协调工具，主要用于分布式锁、服务注册和发现、共享配置和状态信息

### zookeeper节点类型


    -  PERSISTENT  持久化
    -  PERSISTENT_SEQUENTIAL  持久化顺序节点
    -  EPHEMERAL  临时（ 客户端session超时这类节点就会被自动删除 ）
    -  EPHEMERAL_SEQUENTIAL   临时顺序节点

### zookeeper实现分布式锁流程


    - 所有客户端都在指定znode下创建临时顺序节点
    - 取该节点下的所有节点，按照需要排序
    - 查看自己是否是最小的节点
    
      - 如果自己是，则获取到锁
      - 如果自己不是，则监听次小（n-1）节点的删除事件。 次小节点删除即获取到锁

### curator的分布式锁实现

```java
public static void main(String[] args) {

    final String connectString = "localhost:2181,localhost:2182,localhost:2183";

    // 重试策略，初始化每次重试之间需要等待的时间，基准等待时间为1秒。
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

    // 使用默认的会话时间（60秒）和连接超时时间（15秒）来创建 Zookeeper 客户端
    @Cleanup CuratorFramework client = CuratorFrameworkFactory.builder().
        connectString(connectString).
        connectionTimeoutMs(15 * 1000).
        sessionTimeoutMs(60 * 100).
        retryPolicy(retryPolicy).
        build();

    // 启动客户端
    client.start();

    final String lockNode = "/lock_node";
    InterProcessMutex lock = new InterProcessMutex(client, lockNode);
    try {
      // 1. Acquire the mutex - blocking until it's available.
      lock.acquire();
      // OR

      // 2. Acquire the mutex - blocks until it's available or the given time expires.
      if (lock.acquire(60, TimeUnit.MINUTES)) {
        Stat stat = client.checkExists().forPath(lockNode);
        if (null != stat){
          // Dot the transaction
        }
      }
    } finally {
      if (lock.isAcquiredInThisProcess()) {
        lock.release();
      }
    }
  }
```

41. curator锁类型

- **zookeper的实现主要有下面四类:**
     - InterProcessMutex：分布式可重入排它锁
  - InterProcessSemaphoreMutex：分布式不可重入排它锁
  - InterProcessReadWriteLock：分布式可重入读写锁
  - InterProcessMultiLock：将多个锁作为单个实体管理的容器

### zookeeper的选举机制

- 节点状态
     - **Looking** ：选举状态。
  - **Following** ：Follower节点（从节点）所处的状态。
  - **Leading** ：Leader节点（主节点）所处状态。
- 崩溃恢复
-  **Leader election**
  - 选举阶段，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID） 
    -  接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，**投票给目前已知最大的ZXID所属节点。** （ zxid相同就比较myid大小，myid大的作为leader服务器）
    -  每次投票后，服务器都会统计投票数量，判断是否有某个节点得到**半数**以上的投票。如果存在这样的节点，该节点将会成为**准Leader**，状态变为Leading。其他节点的状态变为Following。 
  -  **Discovery** 
  
    - 为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。
    - 所以这一情况，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。
    - 各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。
  -  **Synchronization** 
  
    -  同步阶段，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。 

### 数据写入

-  客户端发出写入数据请求给任意Follower。
-  Follower把写入数据请求转发给Leader。
-  Leader采用**二阶段提交**方式，先发送Propose广播给Follower。
-  Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。
-  Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

### ZXID是啥 


    - zookeeper节点所处理的事务id
    -  最大ZXID也就是节点本地的最新事务编号，包含**epoch**和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term。 

### zookeeper数据模型

- 采用类目录树结构,树是由节点所组成，Zookeeper的数据存储也同样是基于节点，这种节点叫做**Znode**。 
  
- znode数据结构
	- **data**：Znode存储的数据信息。
    - **ACL**：记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。
    - **stat**：包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。
    - **child**：当前节点的子节点引用，类似于二叉树的左孩子右孩子。
  
- 基本操作
	- **create**：创建节点
    - **delete**：删除节点
    - **exists**：判断节点是否存在
    - **getData**：获得一个节点的数据
    - **setData**：设置一个节点的数据
    - **getChildren**：获取节点下的所有子节点
  
  

### 事件通知

- Zookeeper客户端在请求读操作的时候，可以选择是否设置**Watch**。Watch是什么意思呢？
- 我们可以理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知。

## Dubbo

- Dubbo如何处理拆包、粘包问题的？
- 这种问题的处理方式有三种：定长消息、使用特殊结束符、采用消息头（消息头带有消息体长度）+消息体的方式
  - dubbo采用消息头+消息体的方式，并且将其封装成为应用层的dubbo协议
- dubbo是异步的，那么客户端使用rpc调用时如何找到response具体需要对应哪个request？
  - 在进行调用的时候，会生成一个唯一的requestId，客户端会将requestId和所需的callback方法绑定存放到map里，然后等待服务端返回相同requestId的response
- dubbo常见配置项？
  - dubbo:application 当前系统信息；name：名称，owner：
  - dubbo:registry 注册中心；address：地址，check：启动时是否检查
  - dubbo:protocol 使用的协议；name：协议名称，port：端口
  - dubbo:service 提供远程服务；interface：服务接口全名，ref：bean的id，protocol：协议，version：版本
  - bean 需要注册的bean；id：bean的id，class：具体实现类
  - dubbo:reference 引用远程服务；id：远程服务id，interface 远程服务接口全名
- Java RMI调用方式？
- 服务端：
  - 一个实现Remote接口的实现类，并且需要继承UnicastRemoteObject类
    - 使用LocateRegistry.createRegistry(1099);  暴露接口
  -  java.rmi.Naming.rebind("rmi://localhost:1099/hello", hello);  绑定接口实现
  - 客户端：
    - Naming.lookup("rmi://localhost:1099/hello"); 将其强转为相应接口即可
- RMI和RPC的区别？
- RMI协议只能使用java，RPC是通用协议
  - RMI使用对象交换数据，RPC使用XDR格式交换数据
  - RMI协议基于stub，客户端必须有相应签名才能调用服务端接口
- 常见的容错机制？
1. failover：失效转移 
     Fail-Over的含义为“失效转移”，是一种备份操作模式，当主要组件异常时，其功能转移到备份组件。其要点在于有主有备，且主故障时备可启用，并设置为主。如Mysql的双Master模式，当正在使用的Master出现故障时，可以拿备Master做主使用
  
2. failfast：快速失败 
     从字面含义看就是“快速失败”，尽可能的发现系统中的错误，使系统能够按照事先设定好的错误的流程执行
  
3. failback：失效自动恢复 
     Fail-over之后的自动恢复，在簇网络系统（有两台或多台服务器互联的网络）中，由于要某台服务器进行维修，需要网络资源和服务暂时重定向到备用系统。在此之后将网络资源和服务器恢复为由原始主机提供的过程，称为自动恢复
  
4. failsafe：失效安全 
     Fail-Safe的含义为“失效安全”，即使在故障的情况下也不会造成伤害或者尽量减少伤害。维基百科上一个形象的例子是红绿灯的“冲突监测模块”当监测到错误或者冲突的信号时会将十字路口的红绿灯变为闪烁错误模式，而不是全部显示为绿灯。

## 分布式事务

### 2PC/3PC

[分布式一致性算法2PC和3PC](https://blog.51cto.com/11821908/2058651)

### TCC



## 分布式算法

### Paxos

### ZAB (Zookeeper Atomic  Broadcast)

### Raft

[拜占庭将军问题](https://www.jianshu.com/p/8bcef0ca676c)

- 只有当系统中2/3的参与者是正常的，系统才能够正常使用 （N > 3F + 1）
- 以上算法均基于非拜占庭容错的分布式一致性算法，即除考虑消息的丢失、超时、乱序，但不考虑消息被篡改；只有区块链中使用的相关算法PoW和PoX等才能有限解决

### 一致性hash