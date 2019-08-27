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

### Zuul 网关

### Ribbon 负载均衡

1. 常用注解：@LoadBalanced

### Eureka 服务发现

1. 常用注解：@EnableDiscoveryClient
2. 常用配置： 

### Feign 服务调用

1. 常用注解： @EnableFeignClients、@FeignClient

### Hystrix 服务熔断

1. 常用注解： @EnableHystrix、@EnableCircuitBreaker、@HystrixCommand(fallbackMethod = “”)、@EnableHystrixDashboard、@CacheResult

### Apollo 配置中心

1. 常用注解： @EnableApolloConfig、@ApolloConfig、@ApolloConfigChangeListener

## JUC

### AbstractQueuedSynchronizer（AQS）https://www.jianshu.com/p/da9d051dcc3d

- 内部维护了一个CLH实现的FIFO队列，用于线程等待获取资源；
- 又维护了一个volatile的变量state，用于代表资源；
- 资源有两种模式：互斥的（一次只能一个线程使用，如ReentrantLock）和共享的（有限个数线程共同使用，如Semaphore/CountDownLatch）；

### Unsafe

- 为什么是unsafe？
  - 核心方法都是native的，底层直接操作引用和内存，会造成不可知的风险；故只有JDK的类能使用（限制了只有BootStrapClassLoader加载的类可以使用）
- ABA问题的解决：AtomicStampedReference和 AtomicMarkableReference；使用版本号/时间戳解决

## MQ

### RabbitMQ

### RocketMQ

## Zookeeper

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

## 分布式算法

### Paxos

### Raft

### 一致性hash