平安产险-java岗技术面试题参考： 
1. concurrentHashMap数据结构如何保证并发操作 

   - https://www.cnblogs.com/leeego-123/p/12156165.html
   - put时: volatile + CAS + synchronized
   -  如果key的hash在高X位为1，（X为数组长度的二进制-1的最高位），则扩容时是需要变换在Node数组中的索引值的(+ 原数组长度); 如果在高位为0,则仍在原数组位置
   - 迁移时会使用占位node 'MOVED'(hashcode为-1),用来限制对原node链表的写入
   - get: 使用了 *ForwardingNode* 内部类保存,使之可以在迁移过程中也能进行查找
   - resize:  此时sizeCtl变量用来标示HashMap正在扩容，当其准备扩容时，会将sizeCtl设置为一个负数，（例如数组长度为16时,高16位表示数组长度,低16位表示并发线程数)

2. 垃圾收集算法 

   - 复制算法
   - 标记-清理算法
   - 标记-整理算法

3. 垃圾收集器 

   - Serial/Serial Old收集器 是最基本最古老的收集器，它是一个单线程收集器，并且在它进行垃圾收集时，必须暂停所有用户线程。Serial收集器是针对新生代的收集器，采用的是Copying算法，Serial Old收集器是针对老年代的收集器，采用的是Mark-Compact算法。它的优点是实现简单高效，但是缺点是会给用户带来停顿。
   - ParNew收集器 是Serial收集器的多线程版本，使用多个线程进行垃圾收集。
   - Parallel Scavenge收集器 是一个新生代的多线程收集器（并行收集器），它在回收期间不需要暂停其他用户线程，其采用的是Copying算法，该收集器与前两个收集器有所不同，它主要是为了达到一个可控的吞吐量。
   - Parallel Old收集器 是Parallel Scavenge收集器的老年代版本（并行收集器），使用多线程和Mark-Compact算法。
   - CMS（Current Mark Sweep）收集器 是一种以获取最短回收停顿时间为目标的收集器，它是一种并发收集器，采用的是Mark-Sweep算法。
   - G1收集器 是当今收集器技术发展最前沿的成果，它是一款面向服务端应用的收集器，它能充分利用多CPU、多核环境。因此它是一款并行与并发收集器，并且它能建立可预测的停顿时间模型。

4. cms的缺点如何解决 

   1. 回收过程: 初始标记 -> 并发标记 -> 重新标记 -> 并发清理

   - 回收时间长，吞吐量不如Parallel
   - 无法处理浮动垃圾
   - 会产生大量的空间碎片 (标记-清理算法)
   - 并发模式失败：Concurrent model failure

5. spring代理模式

   - 静态代理

   - 动态代理

     - JDK动态代理 -基于接口

       - ```java
         Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces,InvocationHandler h )
         // 实现InvocationHandler.invoke方法
         ```

     - CGLIB动态代理 - 基于继承

       - ```
         MethodInterceptor 实现intercept接口
         通过enhancer.create()来获取代理类
         ```

6. spring加载过程 

   - 

7. spring aop advise区别 

8. mysql执行引擎,mysql优点

   - MyISAM: 优势 – 查询速度快 – 数据和索引压缩问题 – 表级锁 – 数据丢失
   - InnoDB: 优势 – 行级锁 – 事务支持 – 数据安全问题 – 数据文件庞大 – 启动慢 – 不支持FULLTEXT索引
   - 

9. mysql主备同步 

   - 从主的 Binary log 同步到从的relay log,再进行重放
   - https://www.cnblogs.com/hoje/p/11944556.html

10. redis数据结构

    - string
    - hash
    - list
    - set
    - zset

11. redis String的存储

    ```C
    struct sdshdr{
         //记录buf数组中已使用字节的数量
         //等于 SDS 保存字符串的长度
         int len;
         //记录 buf 数组中未使用字节的数量
         int free;
         //字节数组，用于保存字符串
         char buf[];
    }
    ```

    https://www.cnblogs.com/ysocean/p/9080942.html

12. redis hash优化

    - Redis Hash是value内部为一个HashMap，如果该Map的成员数比较少，则会采用类似一维线性的紧凑格式来存储该Map, 即**省去了大量指针的内存开销**，这个参数控制对应在redis.conf配置文件中下面2项：

      **hash-max-zipmap-entries 64 hash-max-zipmap-value 512**    

      当value这个Map内部不超过多少个成员时会采用线性紧凑格式存储，默认是64,即value内部有64个以下的成员就是使用线性紧凑存储，超过该值自动转成真正的HashMap。

      **hash-max-zipmap-value** 含义是当 value这个Map内部的每个成员值长度不超过多少字节就会采用线性紧凑存储来节省空间。

      以上2个条件任意一个条件超过设置值都会转换成真正的HashMap，也就不会再节省内存了，那么这个值是不是设置的越大越好呢，答案当然是否定的，HashMap的优势就是查找和操作的时间复杂度都是O(1)的，而放弃Hash采用一维存储则是O(n)的时间复杂度，如果

      成员数量很少，则影响不大，否则会严重影响性能，所以要权衡好这个值的设置，总体上还是最根本的时间成本和空间成本上的权衡。

13. redis集群

    - redis-cluster 最少6个节点(3主3从)
    - 16383个hash槽 CRC16('key') 
    -  **redis-trib**  集群管理工具

14. 什么是缓存雪崩、缓存穿透、缓存击穿？

    - 缓存雪崩： 短期内大量缓存过期，导致过多请求打到DB
    - 缓存穿透： 查询数据库和缓存中都没有的数据时，每次都要去db扫表查询
    - 缓存击穿： 同一时间大量并发请求同一个DB有但是缓存里没有的数据（该缓存正在构建中）

15. redis数据同步

    - 

16. redis增加节点 
    
    - redis-server ../7002/redis.conf  # 启动节点
    - redis-cli --cluster add-node # 将节点添加到集群中
    - redis-cli --cluster reshard # 重新分配hash槽
    - 分配哈希槽有两种方式：
    
    　　（1）在其他节点拿出适量的哈希槽分配到目标节点：输入all 需要分配给目标节点的哈希槽来着当前集群的其他主节点（每个节点拿出的数量为集群自动决定）
    
    　　（2）在指定的节点拿出指定数量的哈希槽分配到目标节点：done


17. redis 内存淘汰策略

    - noeviction : 当内存超过配置内存时，会返回错误，不会删除任何键。
    - allkeys-lru: 当内存不足以处理新加入的键时，按照lru算法驱逐最久没有使用的键。（建议使用）
    - volitile-lru: 当内存不足以处理新加入的键时，按照lru算法在设置了过期时间的key中，驱逐最久没有使用的键。
    - allkeys-random: 当内存不足以处理新加入的键时，随机驱逐键。（不建议使用）
    - volitile-random: 当内存不足以处理新加入的键时，在设置了过期时间的key中，随机驱逐键。
    - volitile-ttl: 当内存不足以处理新加入的键时，驱逐马上要过期的键。
    - allkeys-lfu: 当内存不足以处理新加入的键时，按照lfu算法驱逐使用频率最低的键。
    - volitile-lfu: 当内存不足以处理新加入的键时，按照lfu算法在设置了过期时间的key中，驱逐使用频率最低的键。redis的默认缓存淘汰策略是 noeviction，我们可以按照上面的说明选择最合适的方案。

18. 线程池参数.线程如何共享 

    - 常用参数
      - corePoolSize：核心线程数量，会一直存在，除非allowCoreThreadTimeOut设置为true
        maximumPoolSize：线程池允许的最大线程池数量
        keepAliveTime：线程数量超过corePoolSize，空闲线程的最大超时时间
        unit：超时时间的单位
        workQueue：工作队列，保存未执行的Runnable 任务
        threadFactory：创建线程的工厂类
        handler：当线程已满，工作队列也满了的时候，会被调用。被用来实现各种拒绝策略。
    - 线程池拒绝策略
      - AbortPolicy：中断抛出异常
      - DiscardPolicy：默默丢弃任务，不进行任何通知
      - DiscardOldestPolicy：丢弃掉在队列中存在时间最久的任务
      - CallerRunsPolicy：让提交任务的线程去执行任务(对比前三种比较友好一丢丢)
    - 线程数据共享
      - 线程间共享: 成员变量,threadLocal
    - 线程如何复用
    - 

19. 内存模型 

    - 堆
    - 方法区
    - 虚拟机栈
    - 本地方法栈
    - 程序计数器

20. 虚拟机栈结构

    - 局部变量表
    - 操作数栈
    - 动态链接
    - 方法返回地址

21. 服务高可用 

    - 集群

    - 分布式
    - HA +keepalived+ 双备

22. 服务横向扩展

    - 横向扩展 scale out 
      - 通过集群/分布式等架构扩展服务能力
    - 纵向扩展 scale up
      - 通过增加服务器性能扩展服务能力

23. 服务扩展带来的问题 

    - 可用性
    - 一致性
    - 分区容错性

24. 分布式事务

    - XA
    - TCC
    - 柔性事务 (本地日志表 + mq)

25. kafka如何做到高吞吐量的

26. kafka原理 

27. rabbitmq exchange作用 

28. springcloud eureka的协议zone的作用

    - region - zone 用于标识机房所属地区,尽量保证服务调用走最近的链路,减少延时

29. eureka的保护策略 

    -  Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期，但是在保护期内如果服务刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。

    - 自我保护模式被激活的条件是：在 1 分钟后，`Renews (last min) < Renews threshold`。

      这两个参数的意思：

      - `Renews threshold`：**Eureka Server 期望每分钟收到客户端实例续约的总数**。
      - `Renews (last min)`：**Eureka Server 最后 1 分钟收到客户端实例续约的总数**。

30. threadlocal 

    - 每个Thread线程内部都有一个Map。
    - Map里面存储线程本地对象（key）和线程的变量副本（value）
    - 但是，Thread内部的Map是由ThreadLocal维护的，由ThreadLocal负责向map获取和设置线程的变量值。
    - threadLocal对象就是放在ThreadLocalMap里的键

31. synchrinized的优化 https://www.cnblogs.com/jyroy/p/11365935.html

    - 偏向锁
    - 轻量锁
    - 重量锁

32. Java对象结构

    - 对象头
      - 无锁: 对象hashcode + 分代年龄 + 偏向锁标志 +锁标志01
      - 偏向锁: 线程id +  分代年龄 + 偏向锁标志 +锁标志01
      - 轻量级锁: 指向栈中锁记录的指针 +锁标志00
      - 重量级锁: 指向重量级锁的指针 +锁标志10
      - GC: 锁标志11
    - 实例数据
    - 对齐填充(保证对象大小是8的整数倍,这样可以支持指针压缩,让32位4字节地址(2^32 = 4G)可以访问32G内存)

33. AQS的实现类，AQS原理 

34. 熔断策略配置了哪些参数? 熔断failback作用 ?


    - 

35. java的锁用过哪些? synchronized实现 


    - 

36. sleep()、wait() 和 park() 的区别

    |              | sleep                        | wait               | park                                  |
    | ------------ | ---------------------------- | ------------------ | ------------------------------------- |
    | 实现方式     |                              |                    |                                       |
    | 线程状态     | timed_waiting                | waiting            | waiting                               |
    | 是否响应中断 | 需要捕获interruptedException | 同左               | 不需要                                |
    | 是否释放锁   | 否                           | 是                 | 否                                    |
    | 唤醒信号     | 超时                         | wait必须先于notify | unpark可以先于park(但是最多只记录1次) |

    

37. zookeeper是什么?

    - zookeeper是一个分布式协调工具，主要用于分布式锁、服务注册和发现、共享配置和状态信息

38. zookeeper节点类型


    -  PERSISTENT  持久化
    -  PERSISTENT_SEQUENTIAL  持久化顺序节点
    -  EPHEMERAL  临时（ 客户端session超时这类节点就会被自动删除 ）
    -  EPHEMERAL_SEQUENTIAL   临时顺序节点

39. zookeeper实现分布式锁流程


    - 所有客户端都在指定znode下创建临时顺序节点
    - 取该节点下的所有节点，按照需要排序
    - 查看自己是否是最小的节点
    
      - 如果自己是，则获取到锁
      - 如果自己不是，则监听次小（n-1）节点的删除事件。 次小节点删除即获取到锁

40. curator的分布式锁实现

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


    - **zookeper的实现主要有下面四类类:**
      - InterProcessMutex：分布式可重入排它锁
      - InterProcessSemaphoreMutex：分布式不可重入排它锁
      - InterProcessReadWriteLock：分布式可重入读写锁
      - InterProcessMultiLock：将多个锁作为单个实体管理的容器

42. zookeeper的选举机制


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
    - 数据写入
    
      - 客户端发出写入数据请求给任意Follower。
      - Follower把写入数据请求转发给Leader。
      - Leader采用**二阶段提交**方式，先发送Propose广播给Follower。
      - Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。
      - Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

43. ZXID是啥 


    - zookeeper节点所处理的事务id
    -  最大ZXID也就是节点本地的最新事务编号，包含**epoch**和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term。 

44. zookeeper数据模型


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
    
    - 事件通知


      - Zookeeper客户端在请求读操作的时候，可以选择是否设置**Watch**。Watch是什么意思呢？
    
        我们可以理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知。

45. 分布式协议知道哪些?


    - paxos
    - raft
    - zab
    - gossip

46. springboot怎么做一个starter 


```java
1. 编写 resources/META-INF/spring.factories 

   - org.springframework.boot.autoconfigure.EnableAutoConfiguration=配置类
2. 添加依赖  spring-boot-autoconfigure 
3. 使用conditional系列依赖 

   -  @ConditionalOnProperty 
   - @ConditionalOnBean
   - @ConditionalOnClass
   - @ConditionalOnMissingBean
```

38. List是怎么实现的 
    - arrayList 数组(增删改O(n),查O(1))
    - linkedlist 链表(增删改O(1),查O(n))
    
39. 根据应用场景考察基础技术问题 jvm（内存调优参数） 
    - 抢单: 因为抢单过后访问量会大幅下降,多数对象都很少复用,所以可以调大年轻代堆大小
    - 调优参数: 栈大小、堆大小、新生代和老年代比例
    -  Java Performance里面的推荐公式设置
      -  Java整个堆大小设置，Xmx 和 Xms设置为老年代存活对象的3-4倍，即FullGC之后的老年代内存占用的3-4倍（ 使用jmap dump:live工具可触发FullGC ）
      - 永久代 PermSize和MaxPermSize设置为老年代存活对象的1.2-1.5倍。
      - 年轻代Xmn的设置为老年代存活对象的1-1.5倍。
      - 老年代的内存大小设置为老年代存活对象的2-3倍。 
      -  Sun官方建议年轻代的大小为整个堆的3/8左右 
    
40. ConcurrentHashMap底层

41. synchronized

42.  cas原理？ CAS会出现哪些问题？
    - cas原理：  CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。 
    - ps: unsafe的获取 反射
    - cas的缺点
      - 循环次数过多，空转占用开销大
      - ABA问题 （ AtomicStampedReference 解决）
      -  **只能保证一个共享变量的原子操作** （ **AtomicReference** 解决）
    
43. 破坏单例的方式
    - 克隆 保护方式：重写clone()方法
    - 反射 保护方式：添加实例计数，计数增加报错
    - 序列化 保护方式：添加readObject方法
    
44. AbstractQueuedSynchronizer(AQS)详解. https://www.cnblogs.com/waterystone/p/4920797.html

    - AQS对象中以CLH队列(FIFO)来进行线程的插入

45. NIO

46. springboot (ioc流程，starter) 

47. 数据库（索引，优化）

48. redis 

49. 常见mysql

50. b＋树红黑树

51. spring ioc设计模式

52. spring bean生命周期

    1. Bean自身的方法。比如构造方法，getter setter方法，其他的自定方法

    2. Bean级的生命周期方法。要调用这些方法需要由Bean来实现Bean级的生命周期方法接口：比如BeanNameAware,BeanFactoryAware,InitializingBean,DisposableBean,这些接口中的方法是自动调用的，作用范围只是针对实现了前面所说的4个接口的类型的Bean

    3. 容器级的生命周期接口方法。图中带有☆的方法就是容器级的方法，容器中所有的bean都要被容器级的生命周期方法处理，而且这个级别的接口是单独实现的，独立于Bean之外。上图中的容器及生命周期接口为：InstantiationAwareBeanPostProcessor 和BeanPostProcessor

    4. 工厂后处理器接口方法：包括AspectJWeavingEnabler,CustomeAutowireConfigurer,ConfigurationClassPostProcessor等方法，在应用上下文装配文件后立即调用

53. volatile关键字 

    - 保证了字段的可见性(修改后可以立即被其它线程/cpu发现),jvm内存屏障保证了volatile的顺序性,防止指令重排序
    - 会出现伪共享,缓存行一次加载64个字节的数据. volatile变量失效时会将整个缓存行失效,导致缓存在同一行的数据也需要同时加载. 参考disruptor的解决方案: 使用7个空long字段填充

54. mybatis一二级缓存实现，xml转换sql过程 

55. 加密算法种类说下区别 

    - 对称加密
      -  DES 
      -  3DES 
      -  AES 
    - 非对称加密
      - 如果使用 **公钥** 对数据 **进行加密**，只有用对应的 **私钥** 才能 **进行解密**。
      - 如果使用 **私钥** 对数据 **进行加密**，只有用对应的 **公钥** 才能 **进行解密**。
      -  **公钥加密、私钥解密、私钥签名、公钥验签。** 
      -  RSA 
      -  DSA 
    - 散列算法
      - SHA-1
      - SHA-256
      - MD5
    - 国密算法
      - SM1 为对称加密。其加密强度与AES相当。该算法不公开，调用该算法时，需要通过加密芯片的接口进行调用。
      - SM2为非对称加密，基于ECC。该算法已公开。由于该算法基于ECC，故其签名速度与秘钥生成速度都快于RSA。ECC 256位（SM2采用的就是ECC 256位的一种）安全强度比RSA 2048位高，但运算速度快于RSA。
      - SM3 消息摘要。可以用MD5作为对比理解。该算法已公开。校验结果为256位。
      - SM4 无线局域网标准的分组数据算法。对称加密，密钥长度和分组长度均为128位。

56. collections.sort算法实现 

    - 是否使用旧版排序
      - 旧版排序
        - 如果小于7 -> 冒泡排序
        - 如果大于等于7 -> 归并排序
      - 新版排序
        - 如果小于32 -> 二叉排序
        - 如果大于等于32 -> 

57. jvm垃圾回收算法过程

58. G1,CMS 

59. mysql数据读取磁盘次数，过程说明下 

    - mysql索引形式
      - 主键采用聚簇索引,叶子节点存储数据
      - 非主键索引采用非聚簇索引(辅助索引),叶子节点存储数据行的主键值,再根据主键回表查询实际数据

60. 锁实现，线程状态，

    - synchronized jvm底层使用ObjectMonitor
    - CAS 使用cpu级 *compxchg* 方法
    - reentrantLock  使用aqs的clh队列实现, Unsafe的park 

61. ObjectMonitor对象的实现? ObjectWaiter对象实现?

    ```C++
    ObjectMonitor::ObjectMonitor() {
      _header       = NULL;
      _count        = 0;
      _waiters      = 0,
      _recursions   = 0;
      _object       = NULL; //锁住的对象
      _owner        = NULL; //当前持有锁的线程
      _WaitSet      = NULL; //所有被调用了wait/park的对象都放在这里面
      _WaitSetLock  = 0 ;
      _Responsible  = NULL ;
      _succ         = NULL ;
      _cxq          = NULL ;
      FreeNext      = NULL ;
      _EntryList    = NULL ; // 调用了notify/notifyAll之后,会有1/n个对象进入此方法
      _SpinFreq     = 0 ;
      _SpinClock    = 0 ;
      OwnerIsThread = 0 ;
    }
    
    class ObjectWaiter : public StackObj {
      ObjectWaiter * volatile _next; //下一个
      ObjectWaiter * volatile _prev; // 上一个
      Thread*       _thread; // 线程
      ParkEvent *   _event;
      volatile int  _notified ;
      volatile TStates TState ; //状态
      Sorted        _Sorted ;           // List placement disposition
      bool          _active ;           // Contention monitoring is enabled
    
    };
    ```

    

62. 线程池创建，

63. 线程waiting, time_waiting区别 

64. Semaphore、CountDownLatCh、 CyclicBarrier的用途区别?

    - CountDownLatch 用于在几个任务同时完成之后再进行后续操作.不可重用
    - CyclicBarrier 用于在几个任务同时完成之后再进行后续操作.可重用
    -  Semaphore 用于控制同时访问的令牌数

65. netty

    1.  [高性能网络通信框架Netty-基础概念篇 - 简书 (jianshu.com)](https://www.jianshu.com/p/5445bb0841d4) 
    2. 影响netty单机连接数的因素
       - 服务器自身配置。内存、CPU、网卡、Linux 支持的最大文件打开数等。
       - 应用自身配置，因为 Netty 本身需要依赖于堆外内存，但是 JVM 本身也是需要占用一部分内存的，比如存放通道关系的大 `Map`。这点需要结合自身情况进行调整。

66. spring如何解决循环依赖的?

    - 三级缓存

67. springboot启动流程简述

    1. 初始化 SpringApplication 应用
       1. 确定是否web环境
       2. 加载 ApplicationContextInitializer 
       3. 加载 ApplicationListener 
    2. 初始化 SpringApplicationRunListeners ,监听启动过程
    3. 加载启动环境,加载配置参数
    4. 加载 ConfigurableApplicationContext 上下文
    5.  prepareContext()方法将listeners、environment、applicationArguments等重要组件与上下文对象关联。 
    6.  refreshContext(context),bean的实例化完成IoC容器可用的最后一道工序。 

68. spring refresh方法简述

    1. 准备refresh
    2. 准备、初始化beanFactory及相关后置处理钩子的加载
    3. 加载bean
    4. 加载完毕

69. 元注解

    1. @Documented 是否支持Java doc
    2. @Retention 注解保留时机 ： 源码 -> class文件 ->运行时
    3. @Target 注解可以作用的位置（类/方法/属性等）
