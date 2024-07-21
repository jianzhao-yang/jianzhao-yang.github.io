# redis基础操作

## redis数据结构

#### redis对象结构体

### 基础结构

#### Db结构

```c
typedef struct redisDb {
    dict *dict;                 // 键空间，存储所有的键值对
    dict *expires;              // 过期键，用于存储设置了过期时间的键
    dict *blocking_keys;        // 阻塞键，用于存储被 BLPOP、BRPOP、BRPOPLPUSH 阻塞的键
    dict *ready_keys;           // 就绪键，用于存储被 UNBLOCK 命令唤醒的键
    dict *watched_keys;         // 监视键，用于事务中的 WATCH 命令
    int id;                     // 数据库编号
    long long avg_ttl;          // 平均 TTL（生存时间），用于统计
    unsigned long expires_cursor;   // 过期键的游标，用于过期键的惰性删除
} redisDb;
```



#### redis中的对象RedisObject

```C
/*
 * Redis 对象 16字节
 */
typedef struct redisObject {
    // 5种数据类型 4bits（0：string，1：list，2：set，3：zset，4: hash）
    unsigned type:4;
    // 数据的编码方式 4bits
    unsigned encoding:4;
    // LRU 时间（相对于 server.lruclock） 24bits,最近一次被访问的时间
    unsigned lru:22;
    // 引用计数 Redis里面的数据可以通过引用计数进行共享 32bits
    int refcount;
    // 指向对象的值 64-bit
    void *ptr;
} robj;// 16bytes
```

#### redis数据的编码方式

![](http://www.yund.tech/yund-cms/sys/common/view/files/20200403/d58a27f7-6cb2-4a2b-bb23-44a9e3801ff7.png)

```
 REDIS_ENCODING_RAW （简单动态字符串）
 REDIS_ENCODING_INT（long 类型的整数）
 REDIS_ENCODING_HT （字典）
 REDIS_ENCODING_EMBSTR embstr （内嵌的简单动态字符串）
 REDIS_ENCODING_LINKEDLIST （双端链表）
 REDIS_ENCODING_ZIPLIST （压缩列表）
 REDIS_ENCODING_INTSET （整数集合）
 REDIS_ENCODING_SKIPLIST （跳跃表和字典）
 REDIS_ENCODING_QUICKLIST (快速表-双端ziplist链表)
 REDIS_ENCODING_STREAM (STREAM流)
```

#### 类型和编码选择

string：int（数字且小于long.max）、embstr(长度小于44)、raw

list：3.2之前：数据量和大小较小时用ziplist，否则linkedlist，3.2之后统一用quicklist

set：intset、ht

zset：ziplist（数量<128,单元素大小<64byte，两个紧邻元素存放element和score）、ht、skiplist

hash: ziplist（数量<512,氮元素<64）、ht

#### ziplist 压缩表

```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```

- zlbytes: zl列表总字节数，32bits
- zltail: zl列表最后一个entry的指针，32bits
- zllen: zl列表entry总数，16bits
- entry: zl列表元素
- zlend: zl列表结束标志，8bits

ziplist元素entry包括三部分内容：

- prevlen：前一项的长度。方便快速找到前一个元素地址，如果当前元素地址是x，(x-prelen)则是前一个元素的地址
- encoding：当前项长度信息的编码结果。
- data：当前项的实际存储数据



##### 简介

- 压缩列表是Redis为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值。
- 压缩列表是一段连续内存组成的空间，与数组相比，**单个元素长度不用保持一致**；与链表相比，连续内存效率更高，且不用保存指针节省了内存
- 用于节点数较少的hash和list

##### 数据结构

- 压缩列表结构
  - zlbytes：记录整个压缩列表占用的内存字节数。
  - zltail：记录压缩列表表尾节点距离压缩列表起始地址有多少字节。（用于快速找到尾节点）
  - zllen：记录了压缩列表包含的节点数量。
  - entryN：压缩列表的数据节点，节点长度由节点保存的内容决定。
  - zlend：特殊值0xFF（十进制255），用于标记压缩列表的末端。
- 数据节点entry结构
  - previous_entry_length 记录上一个节点的长度（用于反向遍历）（如果长度小于254，则1个字节；如果大于254，则用0xfe作为开始标识，后4个字节保存实际长度）
  - encoding 记录节点的数据类型和数据长度
  - contents 记录数据
  
  问题：数据量大时连续内存申请困难；因为每个元素保存了上一个元素长度，如果新插入的元素数据量过大，可能引发连锁更新

#### quicklist

redis源码对quicklist的释义：*A doubly linked list of ziplists* （一个ziplist组成的双端链表）

##### 数据结构

```c
typedef struct quicklist {
    quicklistNode *head;   // quicklist的链表头
    quicklistNode *tail;   // quicklist的链表尾
    unsigned long count;   // 所有ziplist中的总元素个数
    unsigned long len;     // quicklistNodes的个数
    int fill : QL_FILL_BITS;  // （大于0表示每个ziplist最多节点数量，<0时为枚举，默认值-2表示每个ziplist最大8k内存）
    unsigned int compress : QL_COMP_BITS; // 具体含义是两端各有compress个节点不压缩，0为全部不压缩
    ...
} quicklist;

typedef struct quicklistNode {// 包装后的ziplist
    struct quicklistNode *prev;  //前一个quicklistNode
    struct quicklistNode *next;  //后一个quicklistNode
    unsigned char *zl;           //当前node的ziplist指针，预留了扩展，后续可能不止ziplist
    unsigned int sz;             //ziplist的字节大小
    unsigned int count : 16;     //该ziplist中的元素个数
    unsigned int encoding : 2;   //编码格式，1 原生字节数组或 2压缩存储
    unsigned int container : 2;  //存储方式，默认2 使用ziplist，1为预留
    unsigned int recompress : 1; //数据是否被压缩
    unsigned int attempted_compress : 1; //数据能否被压缩
    unsigned int extra : 10;     //预留的bit位
} quicklistNode;

```

特点：

- 是一个ziplist的双端链表
- 继承了ziplist的优势，连续内存，节省空间，且提高访问效率
- 兼具了链表优势，解决了ziplist在数据元素过多时申请连续大内存困难的问题
- 可以压缩ziplist中的元素，节省内存



#### skiplist 跳跃表

特性

- 键值存储
- 键唯一
- 可排序

使用skiplist+dict实现

##### 数据结构

```c
/**
 * 有序集合结构体
 */
typedef struct zset {
    
    // 元素-分值的map，所以元素必须唯一,对于存在的只更新元素分值
    dict *dict;
	//底层指向的跳跃表的指针
    zskiplist *zsl;
} zset;

/**
 * 跳跃表结构体
 */
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

/**
 * ZSETs use a specialized version of Skiplists
 * 跳跃表中的数据节点
 */
typedef struct zskiplistNode {
    sds ele;//数据
    // 分数
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        /**
         * 跨度实际上是用来计算元素排名(rank)的，
         * 在查找某个节点的过程中，将沿途访过的所有层的跨度累积起来，
         * 得到的结果就是目标节点在跳跃表中的排位
         */
        unsigned long span;
    } level[];
} zskiplistNode;
```

- 参考  [死磕Redis5.0之跳跃表 - 简书 (jianshu.com)](https://www.jianshu.com/p/c2841d65df4c) 

  

#### String

- redis中的字符串只能保存字符串，不可以保存二进制内容
- 字符串中不可以有C语言\0的值，因为它是redis字符串结束标识

##### 为什么要自定义一个字符串结构？

C语言字符串问题：

- 获取字符串长度时间复杂度为0（n）
- 非二进制安全（遇到用户自定义‘\0’结束符直接当作末尾）
- 不支持修改

Redis字符串

- 获取长度时间复杂度为0（1），因为有专门保存长度的字段
- 支持动态扩容
- 减少内存分配次数（分配内存=原字符串长度*2+1，大于1M时为原长度+1M+1）
- 二进制安全，以长度len为结束，而不是以\0
- redis字符串兼容c语言，所以仍然有\0结束符，但是它既不作为字符串长度计算，也不作为字符串结束标识



##### StringObject 简单动态字符串

```C
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度 4byte
     int len;
     //记录 buf 数组中未使用字节的数量 4byte（如开始字符串是长度是100，修改后为50，则50-100的空间不会回收或清除，仅记录位置）
     int free;
     //字节数组，用于保存字符串 字节\0结尾的字符串占用了1byte
     char buf[];
}
```

##### 编码方式

- int：8个字节的长整型
- embstr：小于等于39个字节的字符串
- raw：大于39个字节的字符串

##### embstr

embstr 是专门用于保存短字符串的一种优化编码方式，跟正常的字符编码相比，字符编码会调用两次内存分配函数来分别创建 redisObject 和 sdshdr 结构（动态字符串结构），而 embstr 编码则通过调用一次内存分配函数来分配一块连续的内存空间，空间中包含 redisObject 和 sdshdr（动态字符串）两个结构，两者在同一个内存块中。从 Redis 3.0 版本开始，字符串引入了 embstr 编码方式，长度小于 OBJ_ENCODING_EMBSTR_SIZE_LIMIT(39) 的字符串将以EMBSTR方式存储。

**注意**： 在Redis 3.2 之后，就不是以 39 为分界线，而是以 44 为分界线，主要与 Redis 中内存分配使用的是 jemalloc 有关。（ jemalloc 分配内存的时候是按照 8、16、32、64 作为 chunk 的单位进行分配的。为了保证采用这种编码方式的字符串能被 jemalloc 分配在同一个 chunk 中，该字符串长度不能超过64，故字符串长度限制

- 为什么在3.2版本之后,embstr字符串大小限制从39变为了44?
  - jemalloc 内存分配为64个字节一组
  - 3.2时**redis底层将sdshdr拆分为sdshdr5、sdshdr8、sdshdr16、sdshdr32（等同于原sdshdr）、sdshdr64，并且添加一个flag字段标识数据类型**
  - 3.0时，可用str长度为： 64 - redisObject大小（16）- sdshdr大小（free4 + len4 + buf1 = 9）=39
  - 3.2时，可用str长度为： 64 - redisObject大小（16）- sdshdr8大小（free1 + len1 + buf1 +flag1 = 4）=44

#### intset

结构

```c
typedef struct intset {
    uint32_t encoding;// 编码类型
    uint32_t length;// 长度
    int8_t contents[];//内容
} intset;
```

其中，encoding 表示编码，有 int16_t、int32_t 、int64_t 三种取值（对应Java的short、int、long）；length 表示 intset 集合的元素个数；contents 则存储具体的数据。

优点：

- 内存连续，查找时可以用偏移计算
- 存储内容小时使用小编码，可以节省空间
- 编码升级扩容时从最后一个元素开始移位，实现in-place扩容，无需复制数组
- 二分查找确认set是否存在元素，速度快

缺点：

- 内存连续，数据量过大申请大量连续内存困难
- 新加入元素过大时，需要改变编码，并移动所有数据
- 仅支持number存入，新存入string类型时需要更换数据结构

#### Dict 字典

结构

```c
typedef struct dict {// 顶层结构
    dictType *type; // 该字典对应的特定操作函数
    void *privdata; // 该字典依赖的数据
    dictht ht[2]; // hash 表，存储真实数据,另一个rehash时用
    long rehashidx; /* 默认 rehashidx == -1 表明前没有进行rehashing  */
    unsigned long iterators; /* 当前运行的迭代器数 */
} dict;
typedef struct dictht {// hash表，结构类似java hashmap
    dictEntry **table;// 数组
    unsigned long size;// 数组大小，必须是2*n
    unsigned long sizemask;// 恒等于size-1,用于做item.hashcode & sizemask运算,计算出元素所在table[]位置
    unsigned long used;// 记录存储元素个数
} dictht;
typedef struct dictEntry {// kv对
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

```

为什么ht[2]要有两个hash表结构？ 在线扩容时用（不阻塞）

什么时候进行rehash？

- 服务器未运行bgsave | bgrewriteaof,负载因子 = used/size >=1，执行扩容到used下一个2的N次方
- 服务器运行了bgsave | bgrewriteaof,负载因子 = used/size >=5(dict_force_resize_ratio)，执行扩容到used下一个2的N次方
- 负载因子<0.1,执行缩容

怎么rehash？

- 在确定要进行rehash时没有实际执行，只是创建了一个新的ht，并将rehashidx改为0，在每次增删改时处理对应table[]角标的一个链表
- rehash期间，新增只发生在新ht里，删改查同时发生在两个ht
- 

## 特殊数据结构

### stream

https://zhuanlan.zhihu.com/p/60501638

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

## 其他问题

### 渐进式rehash

redis是使用hash表来保存所有键值的，那么它就一定会遇到hash表冲突的问题，那么它是如何处理的呢？

1. 系统中会有两个hash表h0,和h1；平时只有h0有值，在需要rehash时2者都有值
2. 开始rehash时会设置h1大小为h0的2倍
3. rehash期间所有的查找、修改和删除都需要在h0和h1中进行操作，新增只在h1进行
4. rehash时，逐步将h0中的每个桶的值搬运到h1中

### redis到底有几个线程

- 主线程，命令处理，操作数据的
- 后台线程-aof重写（单个操作aof写仍在主线程）
- 后台线程-释放内存 unlink（客户端操作只标识不再使用，后台异步释放内存）
- 网络线程-网络连接（5.0后）

### 为什么要使用单线程

- 纯内存操作，速度快；性能问题在于网络，多线程对此无影响
- 多线程会导致上下文频繁切换
- 多线程会产生并发线程安全问题，实现复杂度高，且引入锁后性能会大幅下降。

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