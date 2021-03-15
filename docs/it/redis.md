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

- https://www.cnblogs.com/-wenli/p/12777703.html
- 使用lpush brpop实现
- 使用PUB/SUB的方式，缺点：如果发布消息时消费者不在线则直接丢失消息