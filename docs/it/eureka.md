# Eureka

## 原理

![](images/eureka_02.png)

### 注册表存储

- eureka中使用双层 ConcurrentHashMap 进行数据存储. 存储为registry

- ```java
  private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry= new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
  ```

- 第一层的 ConcurrentHashMap 的 `key=spring.application.name` 也就是客户端实例注册的应用名；value 为嵌套的 ConcurrentHashMap。

- 第二层嵌套的 ConcurrentHashMap 的 `key=instanceId` 也就是服务的唯一实例 ID，value 为 Lease 对象，Lease 对象存储着这个实例的所有注册信息，包括 ip 、端口、属性等。

### 缓存设计

![缓存设计](images/eureka_03.png)

#### eureka server缓存

- 一级数据,两级缓存  **registry、readWriteCacheMap、readOnlyCacheMap** 
  - registry 用于保存所有注册的服务
  - readWriteCacheMap 过期时间180s,60s从registry定时同步,处理registry中的过期服务
  - readOnlyCacheMap 30s从readWriteCacheMap 定时同步,用于向client提供注册表

#### eureka client缓存

-  DiscoveryClient 中 localRegionApps 从server拉取信息
-  Ribbon服务中的LoadBalancerStats. upServerListZoneMap没30s从client中拉取在线服务 

#### 缓存导致服务延时问题

| Eureka Client | 时间                                       | 说明                                                         |
| ------------- | ------------------------------------------ | ------------------------------------------------------------ |
| 上线          | 30(readOnly)+30(Client)+30(Ribbon)=**90s** | readWrite -> readOnly -> Client -> Ribbon 各30s              |
| 正常下线      | 30(readonly)+30(Client)+30(Ribbon)=**90s** | 服务正常下线（kill或kill -15杀死进程）会给进程善后机会，DiscoveryClient.shutdown()将向Server更新自身状态为DOWN，然后发送DELETE请求注销自己，registry和readWriteCacheMap实时更新，故UI将不再显示该服务实例 |
| 非正常下线    | 30+60(evict)*2+30+30+30= **240s**          | 服务非正常下线（kill -9杀死进程或进程崩溃）不会触发DiscoveryClient.shutdown()方法，Eureka Server将依赖每60s清理超过90s未续约服务从registry和readWriteCacheMap中删除该服务实例 |



## 特性

### 集群

-  Eureka各个节点都是平等的，几个节点挂点不会影响节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka得客户端在向某个Eureka注册得时候如果发现了连接失败，则会自动切换至其他节点，只要有一台Eureka还存活，就能保证注册服务得可用性（即保证可用性），只不过查询的信息不是最新的（不能保证强一致性） 

### 自我保护

-  Eureka还有一种自我保护机制，如果在**15分钟内超过85%**的节点都没要正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况。
    　　1. **不移除**过期服务. Eureka不再从注册列表中移除因为长时间没有收到心跳而应该过期的服务。
      　　2. **不同步**新注册服务到其它节点. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其他节点上（即保证了当前节点依然可用）。
        　　3. **恢复后同步**. 当网络稳定的的时候，当前实例新的注册信息会被同步到其他节点中。
      
      　　因此，Eureka可以很好的应对因为网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。 
  
  - 关闭自我保护  eureka.server.enable-self-preservation=false

### 服务分区

- eureka提供了完整的两地三中心式的高可用方案. 具体为region标识多地,zone标识每一个中心

- 指定eureka服务调用可选和优选地区

```yaml
eureka:
  client:
    prefer-same-zone-eureka: true # 尽可能向同一区域的 eureka 注册,减少延时,默认为true
    region: huabei    #地区
    availability-zones:
      huabei: zone-1,zone-2
      # huanan: zone-3 #可能有多个地区,其它地区作为备选服务
    service-url:
      zone-1: http://localhost:30000/eureka/
      zone-2: http://localhost:30001/eureka/
```

- 指定自己是哪个地区

```yaml
eureka:
  instance:
    metadata-map:
      zone: zone-1
```

## 问题

### eureka所有节点都失效服务之间还能互相调用吗?

- 可以,客户端有缓存

### eureka上注册的服务临时重启(服务不可用,但是还在注册中心)会发生什么?

- 服务调用失败
- 可以配置ribbon调用其它可用服务

