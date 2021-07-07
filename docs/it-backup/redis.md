# redis常见问题汇总

## redis数据结构

#### redis对象结构体

### 基础结构

#### redis中的对象

```C
/*
 * Redis 对象
 */
typedef struct redisObject {
    // 5种数据类型 4bits
    unsigned type:4;
    // 数据的编码方式 4bits
    unsigned encoding:4;
    // LRU 时间（相对于 server.lruclock） 24bits
    unsigned lru:22;
    // 引用计数 Redis里面的数据可以通过引用计数进行共享 32bits
    int refcount;
    // 指向对象的值 64-bit
    void *ptr;
} robj;// 16bytes
```

#### redis数据的编码方式

```
 REDIS_ENCODING_INT（long 类型的整数）
 REDIS_ENCODING_EMBSTR embstr （编码的简单动态字符串）
 REDIS_ENCODING_RAW （简单动态字符串）
 REDIS_ENCODING_HT （字典）
 REDIS_ENCODING_LINKEDLIST （双端链表）
 REDIS_ENCODING_ZIPLIST （压缩列表）
 REDIS_ENCODING_INTSET （整数集合）
 REDIS_ENCODING_SKIPLIST （跳跃表和字典）
```

#### StringObject 简单动态字符串

```C
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度 4byte
     int len;
     //记录 buf 数组中未使用字节的数量 4byte
     int free;
     //字节数组，用于保存字符串 字节\0结尾的字符串占用了1byte
     char buf[];
}
```

#### ziplist

##### 简介

- 压缩列表是Redis为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值。
- 压缩列表是一段连续内存组成的空间，与数组相比，单个元素长度不用保持一致；与链表相比，连续内存效率更高，且不用保存指针节省了内存
- 用于节点数较少的hash和list

##### 数据结构

- 压缩列表结构
  - zlbytes：记录整个压缩列表占用的内存字节数。
  - zltail：记录压缩列表表尾节点距离压缩列表起始地址有多少字节。（用于快速找到尾节点）
  - zllen：记录了压缩列表包含的节点数量。
  - entryN：压缩列表的数据节点，节点长度由节点保存的内容决定。
  - zlend：特殊值0xFF（十进制255），用于标记压缩列表的末端。
- 数据节点entry结构
  - previous_entry_length 记录上一个节点的长度（用于反向遍历）
  - encoding 记录节点的数据类型和数据长度
  - contents 记录数据



### String

#### 编码方式

- int：8个字节的长整型
- embstr：小于等于39个字节的字符串
- raw：大于39个字节的字符串

#### embstr

embstr 是专门用于保存短字符串的一种优化编码方式，跟正常的字符编码相比，字符编码会调用两次内存分配函数来分别创建 redisObject 和 sdshdr 结构（动态字符串结构），而 embstr 编码则通过调用一次内存分配函数来分配一块连续的内存空间，空间中包含 redisObject 和 sdshdr（动态字符串）两个结构，两者在同一个内存块中。从 Redis 3.0 版本开始，字符串引入了 embstr 编码方式，长度小于 OBJ_ENCODING_EMBSTR_SIZE_LIMIT(39) 的字符串将以EMBSTR方式存储。

**注意**： 在Redis 3.2 之后，就不是以 39 为分界线，而是以 44 为分界线，主要与 Redis 中内存分配使用的是 jemalloc 有关。（ jemalloc 分配内存的时候是按照 8、16、32、64 作为 chunk 的单位进行分配的。为了保证采用这种编码方式的字符串能被 jemalloc 分配在同一个 chunk 中，该字符串长度不能超过64，故字符串长度限制

- 为什么在3.2版本之后,embstr字符串大小限制从39变为了44?
  - jemalloc 内存分配为64个字节一组
  - 3.2时redis底层将sdshdr拆分为sdshdr5、sdshdr8、sdshdr16、sdshdr32（等同于原sdshdr）、sdshdr64，并且添加一个flag字段标识数据类型
  - 3.0时，可用str长度为： 64 - redisObject大小（16）- sdshdr大小（free4 + len4 + buf1 = 9）=39
  - 3.2时，可用str长度为： 64 - redisObject大小（16）- sdshdr8大小（free1 + len1 + buf1 +flag1 = 4）=44

## 特殊数据结构

### HyperLogLog 

-  HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。 
- 操作命令
  - pfadd key value 向set集合中添加一个元素
  - pfcount key 计算set集合中元素个数(不重复)
  - pfmerge key1 key2 ... 将n个集合聚合为一个
- 适用场景
  - 去重计数

### 布隆过滤器

- 安装
  - 下载编译布隆过滤器 https://github.com/RedisLabsModules/redisbloom/
  - redis配置文件指定加载布隆过滤器  loadmodule /usr/local/cluster/redisbloom/RedisBloom-1.1.1/rebloom.so  
- 使用方式
  - bf.add key value 向集合中添加值
  - bf.exists  key value 判断值是否在集合中(不存在100%准确,存在近似100%准确)
- 适用场景
  - 重复值判断
  - 缓存击穿问题
  - 反垃圾邮件

### bitMap位图

- 使用方式

  -  getbit key offset   用于获取Redis中指定key对应的值，中对应offset的bit 
  -  setbit key offset value  用于修改指定key对应的值，中对应offset的bit 
  -   bitcount key [start end]   用于统计字符串被设置为1的bit数 **(start和end是byte而不是bit !!!)**
  -  bitop and/or/xor/not destkey key [key …]  用于对多个key求逻辑与/逻辑或/逻辑异或/逻辑非 

- bitcout的陷阱

  如果你有仔细看前文的用法，会发现有这么一个备注“返回一个指定key范围中的值为1的个数(是以byte为单位不是bit)”，这就是坑的所在。1byte = 8bit

- 适用场景

  - 用户的某个属性开关设置
  - 用户在线状态
  - 用户签到状态

- bitmap和布隆过滤器比较

  - 布隆过滤器不能确定一个值是否真的存在,而bitmap可以
  - 布隆过滤器空间利用率比bitmap高,一个位可以代表多个值
  - **布隆过滤器可以输入字符串,而bitmap必须转换为数值才可以.**

## redis 集群

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

### 一致性hash和hash槽

### 常用命令

### 选举机制

### 通信机制

## redis实际场景应用

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

- 使用lpush brpop实现
- 使用PUB/SUB的方式，缺点：如果发布消息时消费者不在线则直接丢失消息

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

### 消息队列

https://www.cnblogs.com/-wenli/p/12777703.html

### LRU & LFU 

### 容灾

 [redis 集群如何保证不丢失数据？ – Jack's Blog (liritian.com)](https://www.liritian.com/archives/759) 

### 特殊数据结构

 [Redis HyperLoglog、Bloom Filter 、Bitmap – Jack's Blog (liritian.com)](https://www.liritian.com/archives/714) 

 [一看就懂系列之 详解redis的bitmap在亿级项目中的应用_咖啡色的羊驼-CSDN博客](https://blog.csdn.net/u011957758/article/details/74783347) 