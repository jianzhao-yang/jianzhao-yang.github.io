# redis集群

## 缓存问题

### 缓存雪崩

- 现象： 短期内大量缓存过期，导致过多请求打到DB
- 解决方式： key的过期时间添加随机数

### 缓存击穿

- 现象： 同一时间大量并发请求同一个DB有但是缓存里没有的数据（该缓存正在构建中）
- 解决方式：2级缓存，加锁，后台自动刷新缓存，限流，热点key永不过期

### 缓存穿透

- 现象：查询数据库和缓存中都没有的数据时，每次都要去db扫表查询
- 解决方式：使用布隆过滤器预加载，查询时限定范围，不存在的数据使用null值保存在缓存中

## 数据更新机制



## redis持久化

### RDB

- 配置文件 save t i t秒内有i个key变化则进行持久化
- 

### AOF

- 配置文件 

  ```
  appendonly yes  #开启AOF模式 
  appendfilename "appendonly.aof" #保存数据的AOF文件名称 
  
  # appendfsync always
  appendfsync everysec    #fsync模式,刷盘机制（将修改从缓冲区刷到磁盘）
  # appendfsync no
  
  no-appendfsync-on-rewrite no    #原文3
  
  auto-aof-rewrite-percentage 100 # 自上次aof之后增长的百分比
  auto-aof-rewrite-min-size 64mb  #aof重写最小大小
  
  aof-load-truncated yes  # 是否加载尾部有问题的aof文件（因为断电等导致损坏）
  ```

- aof文件格式

  - ```
    *3
    $3
    set
    $3
    we3
    $4
    1234
    ```

  - 第一行 *3代表有3个参数

  - 第二行 $3代表第一个参数长度为3

  - 第三行 第一个参数

  - 4~7同理

### 混合模式

- 4.0之后新增混合模式，aof文件中保存rdb二进制数据+少量aof
- 体积小，可读性差

- aof-use-rdb-preamble true

## redis 集群

### redis-sentinel



### redis同步机制

#### 全量同步 SYNC
- Redis全量复制一般发生在Slave初始化阶段（SYNC），这时Slave需要将Master上的所有数据都复制一份。具体步骤如下：
  1. 从服务器连接主服务器，发送SYNC命令；
  2. 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令；
  3. 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令；
  4. 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照；
  5. 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令；
  6. 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

- 全量同步条件
  - slave 初始化阶段
  - slave 进行增量同步失败，或者slave断线

#### 全量同步 PSYNC

- 为了优化断线导致的增量同步，redis 2.8之后将sync改为psync，增加了部分同步的功能.
  - 复制积压缓冲区  
    - 复制积压缓冲区是由Master维护的一个固定长度的FIFO队列，它的作用是缓存已经传播出去的命令。当Master进行命令传播时，不仅将命令发送给所有Slave，还会将命令写入到复制积压缓冲区里面。
- psync执行流程
  1. 客户端向服务器发送SLAVEOF命令，让当前服务器成为Slave  
  2.  当前服务器根据自己是否保存Master runid来判断是否是第一次复制，如果是第一次同步则跳转到3，否则跳转到4；  
  3.  向Master发送PSYNC ? -1 命令来进行完整同步；  
  4.  向Master发送PSYNC runid offset；  
  5. Master接收到PSYNC 命令后首先判断runid是否和本机的id一致，如果一致则会再次判断offset偏移量和本机的偏移量相差有没有超过复制积压缓冲区大小，如果没有那么就给Slave发送CONTINUE，此时Slave只需要等待Master传回失去连接期间丢失的命令；  
  6. 如果runid和本机id不一致或者双方offset差距超过了复制积压缓冲区大小，那么就会返回FULLRESYNC runid offset，Slave将runid保存起来，并进行完整同步。

#### 增量同步
1. 从服务器向主服务器发送PSYNC命令，携带主服务器的runid和复制偏移量；
2. 主服务器验证runid和自身runid是否一致，如不一致，则进行全量复制；
3. 主服务器验证复制偏移量是否在积压缓冲区内，如不在，则进行全量复制；
4. 如都验证通过，则主服务器将保持在积压区内的偏移量后的所有数据发送给从服务器，主从服务器再次回到一致状态。

## redis-cluster

### Gossip协议

> redis 维护集群元数据采用的是gossip 协议，**所有节点都持有一份元数据**，不同的节点如果出现了元数据的变更，就不断将元数据发送给其它的节点，让其它节点也进行元数据的变更。
>
> - **优点**
>
> 元数据的更新比较**分散**，不是集中在一个地方，降低了压力；
>
> - **缺点**
>
> 元数据的更新有**延时**，可能导致集群中的一些操作会有一些滞后。

#### 数据类型

- **meet**：某个节点在内部发送了一个gossip meet 消息给**新加入的节点**，通知那个节点去加入我们的集群。然后新节点就会加入到集群的通信中

- **ping**：每个节点都会频繁给其它节点发送 ping，其中包含自己的状态还有自己维护的集群元数据，互相通过 ping **交换元数据**。
- **pong**：ping 和 meet消息的**返回响应**，包含自己的状态和其它信息，也用于信息广播和更新。
- **fail**：某个节点判断另一个节点 fail 之后，就发送 fail 给其它节点，通知其它节点说这个节点已宕机。

#### MEET

![](https://img2020.cnblogs.com/blog/1816118/202012/1816118-20201203211748069-924817932.png)

#### PING/PONG

![](https://img2020.cnblogs.com/blog/1816118/202012/1816118-20201203211735841-445908647.png)

#### PFAIL/FAIL

- PFAIL疑似下线.当一个节点A认为其它节点B下线之后,会将其标记为 *Possible failure* ,通知给其它节点
- 节点C接收到A的通知,会将B加入到自己的下线报告中 (Fail Report) . 
- 节点C尝试对下线报告中的节点B进行通信,如果无法通信也将其标记为PFAIL,并通知给其它节点.
- 所有节点接收到下线报告都会进行客观下线判断: 节点中超过一半(包括自己)的节点都将其标记为下线.则为客观下线
- 立即将客观下线FAIL通知给集群内**所有节点**
- 下线判断周期为 cluster_node_timeout*2 .超过周期不再进行处理

#### 元数据

Redis Cluster 中的每个节点都**维护一份自己视角下的当前整个集群的状态**，主要包括：

1.  当前集群状态
2. 集群中各节点所负责的 slots信息，及其migrate状态
3. 集群中各节点的master-slave状态
4. 集群中各节点的存活状态及怀疑Fail状态

#### 继续深入剖析ping消息

- ping 时要携带一些元数据，如果很频繁，可能会加重网络负担。因此，一般每个节点每秒会执行 10 次 ping，每次会选择 5 个**最久没有通信**的其它节点。
- 当然如果发现某个节点通信延时达到了 **cluster_node_timeout / 2，那么立即发送 ping**，避免数据交换延时过长导致信息严重滞后。比如说，两个节点之间都 10 分钟没有交换数据了，那么整个集群处于严重的元数据不一致的情况，就会有问题。所以 cluster_node_timeout 可以调节，如果调得比较大，那么会降低 ping 的频率。
- 每次 ping，会**带上自己节点的信息，还有就是带上 1/10 其它节点的信息**，发送出去，进行交换。至少包含 3 个其它节点的信息，最多包含 总节点数减 2 个其它节点的信息。

#### 故障迁移

> 当节点的的两个子节点接收到其主节点的FAIL状态消息时，两个节点就会开始发起故障迁移，竞选成为新的Master节点。两个节点参与竞选之前，首先要检查自身是否有资格参与竞选。

##### 资格检查

> 通过子节点最近一次与主节点通信的时间判断是否有选举权限,以及选举优先级

##### 休眠时间计算

> 休眠时间有三部分: 固定时间500ms + 随机时间(0~500ms) + 排名时间(rank * 1000)

##### 选举

- 先唤醒的节点会与其它主节点进行通信,要求将自己选举为主节点
- 一半以上主节点投票给自己之后,将自己替换为主节点,负责原主节点下的slot,通知给其它节点
- 原先的主节点以及该主节点的其他节点自动成为新的主节点的Slave节点。

### 集群搭建

- cluster模式 自动搭建

  -  redis-cli –cluster check ip:port (可指定多个)  –cluster-replicas 1 

- redis-trib搭建

- 纯手动搭建

  1. 修改配置文件
     - 这行注释掉 #bind 127.0.0.1 (注:bind 含义: 绑定本机特定网卡ip,只接受此网卡的请求)
     - port 7001  修改端口
     - pidfile /var/run/redis_7001. *pid 设置 Redis 实例 pid 文件
     - cluster-enabled yes * 启动集群模式
     - cluster-config-file nodes-7001.conf * 这个是集群配置文件 自动建立的文件
     - cluster-node-timeout 15000 * 设置当前节点连接超时毫秒数
  2. 启动redis  ./redis-server redis.conf 
  3. 执行集群发现节点
     -  redis-cli -p 7001 进入一个节点
     -  cluster meet ip port 和一个节点一起组成集群
     - 或者可以使用 redis-cli --cluster add-node # 将节点添加到集群中
     -  cluster nodes 查看集群内节点状态信息
  4. 分配槽位
     -  cluster info 检查节点状态 (fail : 因为还没有分配槽位,无法存放数据)
     - ./redis-cli -p 7001 cluster addslots {0..5461}  给某个节点分配槽位
     -   cluster info  (ok: 可以开始存放数据)
  5. 配置主从
     - ./redis-cli -p port 进入节点
     -  cluster replicate node-id 将当前节点设置为指定node-id(master) 的slave
  6. 删除节点
     -  cluster forget node-id 
  7. 重新分配hash槽
     - redis-cli --cluster reshard # 重新分配hash槽

- redis-trib

  - ```
    ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
    127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
    ```



### 一致性hash

redis集群将所有数据切片成16384个hash槽位进行管理，某个key的数据会分布于其中某个槽位上，而一个实例上会有1到多个hash槽；这样做到目的有两个：1是使集群中的数据分布更加均匀，2是在集群扩缩容时数据的重新分配量更小

### hash槽

### 集群寻址

客户端在连接到集群中某个redis实例时，实例会将redis集群槽位信息发送给客户端，客户端进行缓存。客户端在操作时会根据key做CRC16 hash运算得到key所属槽位信息；再根据槽位信息找到对应实例；

实例之间会共享每个实例的槽位信息，这样每个实例就可以知道所有的槽位分布；当客户端发送一个key的槽位不在当前实例时，会向客户端返回MOVED命令，告诉客户端实际槽位所在的实例位置,此时客户端需要更新槽位信息；当集群正在扩缩容，槽位正在迁移期间，客户端接受到请求并且key已经迁移时，会向客户端发送ASK命令，告诉槽位指向的新实例位置，但是此时客户端不需要更新槽位信息（因为数据未完全迁移，旧实例上的key仍需要接受操作命令；

### 常用命令

### 选举机制 - 哨兵模式

redis选主

redis主节点一开始启动时由配置指定，主节点宕机后由哨兵进行选择，选择规则为：

1. 配置设置的优先级最高的节点优先；
2. 同步offset最大的优先；
3. runid最小的优先；
4. 另外，网络不稳定的不参与选举（）

哨兵选举

### 通信机制

## redis实际场景应用

### 热点key

- 热点key的发现
  - 使用proxy或二级缓存发现热点key
  - facebook的redis-faina进行redis分析
  - redis-cli -hotkeys分析
  - https://blog.csdn.net/weixin_39723655/article/details/111296590

### redis实现top k

- 利用zadd key scoreN memberN添加元素
- 利用zincrby key increment memberN 提高某个元素分值
- 利用zrange key start stop 获取指定排行范围的member
- 利用zrank key member获取指定member的排名
- 利用zremrangebyscore key min max 删除指定分数区间的member
- 利用zremrangebyrank key start stop 删除指定排行范围的member

### redis监听key过期

- https://www.cnblogs.com/yuluoxingkong/p/10475355.html

- 配置文件打开 CONFIG set notify-keyspace-events Ex ，表示需要监听所有key的失效事件
- 失效的key会被发送 publish keyevent@0__:expired sampleKey到管道中
- 通过pub/sub方式监听keyevent@0__:expired 管道中的消息即可
- java中继承spring data redis中的KeyExpirationEventMessageListener 即可处理key的过期事件

### redis实现分布式锁

- 使用**SET key value [EX seconds] [PX milliseconds] [NX|XX]** 命令实现加锁，为了防止锁不被释放，key的设置和过期一定要再同一条命令中

-  使用脚本实现解锁，防止解了别人的锁

- ```C
  if redis.call("get",KEYS[1]) == ARGV[1] // 检查value，确定是自己加的锁
  then
      return redis.call("del",KEYS[1])  // 在一条命令中，防止并发操作上一条检查和解锁中间有其他操作插队
  else
      return 0
  end
  ```

### redis实现滑动窗口

- 使用zset实现,假设使用lq作为key，maxcount为m，窗口大小t秒，当前时间now
  1. 先通过zRemRangeByScore lq 0 (t * 1000)删除不在窗口内的成员
  2. 通过zcard limitq 获取在窗口内的总个数n
  3. 如果m>n,则获取成功；使用zadd lq now now 记录此次请求，返回0
  4. 如果m<=n,则获取失败，返回1

- ```C
  --KEYS[1]:该次限流对应的key
  --ARGV[1]:一分钟之前的时间戳
  --ARGV[2]:此时此刻的时间戳
  --ARGV[3]:允许通过的最大数量
  --ARGV[4]:member名称（随机生成）
  redis.call('zremrangeByScore', KEYS[1], 0, ARGV[1])  // 1. ZREMRANGEBYSCORE key min max,删除窗口外成员
  local res = redis.call('zcard', KEYS[1]) //2. 获取当前成员计数
  if (res == nil) or (res < tonumber(ARGV[3])) then //3. 判断计数是否大于限流值
      redis.call('zadd', KEYS[1], ARGV[2], ARGV[4])  // 	4. ZADD key score1 member1，不大于则添加
      return 0
  else //4
      return 1
  end
  ```

### redis实现排行榜

- 使用ZINCRBY key increment member为指定key的指定成员添加分数
- 使用 ZREVRANGE key start stop [WITHSCORES] 获取指定排名范围的序列

### redis实现消息队列

- 使用lpush brpop实现；消息确认使用brpoplpush source dest timeout将消息弹出并压入另一个队列（专门用来做消息确认）
- 使用PUB/SUB的方式，缺点：如果发布消息时消费者不在线则直接丢失消息
- stream方式，5.0新增



## key过期删除机制

- redis使用惰性删除+定时删除的策略
- 惰性删除：当使用到key的时候检测key是否过期，如果过期返回nil
- 定时删除：定时任务随机抽取20个key，删除过期的key，如果过期的key超过5个，则重复操作；删除过期key时间片不超过总定时任务时间的25%

## 内存淘汰机制

### 内存淘汰策略

- noeviction : 当内存超过配置内存时，会返回错误，不会删除任何键。
- allkeys-lru: 当内存不足以处理新加入的键时，按照lru算法驱逐最久没有使用的键。（建议使用）
- volitile-lru: 当内存不足以处理新加入的键时，按照lru算法在设置了过期时间的key中，驱逐最久没有使用的键。
- allkeys-random: 当内存不足以处理新加入的键时，随机驱逐键。（不建议使用）
- volitile-random: 当内存不足以处理新加入的键时，在设置了过期时间的key中，随机驱逐键。
- volitile-ttl: 当内存不足以处理新加入的键时，驱逐马上要过期的键。
- allkeys-lfu: 当内存不足以处理新加入的键时，按照lfu算法驱逐使用频率最低的键。
- volitile-lfu: 当内存不足以处理新加入的键时，按照lfu算法在设置了过期时间的key中，驱逐使用频率最低的键。redis的默认缓存淘汰策略是 noeviction，我们可以按照上面的说明选择最合适的方案。

### LRU( Least Recently Used ) 最近时间使用过

#### 标准LRU

- 维护一个链表, 将所有设置了过期时间的 key 放在这个链表中
- 当字典中某个元素被访问时, 它在链表中的位置会被移动到链表头部
- 当空间满时, 就从链表尾部开始移除元素

#### 手动实现LRU

- 使用单向链表实现的问题
  - 查找复杂度O(n),解决方案: 使用一个hashMap存储所有key,时间换空间
  - 将访问的key移动到链表头部时,无法快速连接当前位置的前驱节点和后驱节点. 解决方案: 使用双向链表
- 故应该使用双向链表 + hashMap来存储LRU,这样会导致内存占用急剧增大,所以redis并未采用

#### redis的近似LRU实现

- 3.0之前
  - 给每个 key 增加一个额外 24bit 长度的小字段, 存储该 key 的最后一次访问时间戳
  - 当空间满时, 随机采样取出 5 个 key (数量可配置), 按时间戳淘汰掉最旧的 key
  - 循环第二步, 直到内存低于 maxmemory 值
  - 随机采样的范围取决于配置的策略是 `volatile` 还是 `allkeys`
- 3.0之后
  -  Redis 3.0 开始, 增加了 `淘汰池` 进一步提升了近似 LRU 的效果:
  - 上一次随机采样后未淘汰的 key, 会放入 `淘汰池` 留待下一次循环,
  - 下一次随机采样的 key 会先和 `淘汰池` 中的 key 合并后, 再计算淘汰最旧的 key 

#### redis 的内存回收参数

- maxmemory 10GB 指定最大内存,最好不要超过10G
- maxmemory-policy allkeys-lru/volatile-lru 指定过期策略
- maxmemory-samples 5 指定LRU采样数

### LFU( Least Frequently Used ) 最近访问频率



## redis容灾

### 配置参数

- min-replicas-to-write 1 设置最少slave数量
- min-replicas-max-lag 10 设置slave最大延迟

## 参考博客

### 数据结构

https://www.cnblogs.com/jstarseven/p/12586147.html

https://www.cnblogs.com/hunternet/p/11306690.html

### 主从同步

https://blog.csdn.net/u012538947/article/details/80601356

### redis-cluster

https://www.cnblogs.com/yufeng218/p/13688582.html	

 [Redis-cluster 纯手工搭建 – Jack's Blog (liritian.com)](https://www.liritian.com/archives/737) 

 [集群教程 — Redis 命令参考 (redisdoc.com)](http://redisdoc.com/topic/cluster-tutorial.html) 

gossip协议和故障迁移  https://zhuanlan.zhihu.com/p/106110578

### 消息队列

https://www.cnblogs.com/-wenli/p/12777703.html

### LRU & LFU 

### 容灾

 [redis 集群如何保证不丢失数据？ – Jack's Blog (liritian.com)](https://www.liritian.com/archives/759) 

### 特殊数据结构

 [Redis HyperLoglog、Bloom Filter 、Bitmap – Jack's Blog (liritian.com)](https://www.liritian.com/archives/714) 

 [一看就懂系列之 详解redis的bitmap在亿级项目中的应用_咖啡色的羊驼-CSDN博客](https://blog.csdn.net/u011957758/article/details/74783347) 