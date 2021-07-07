## HashMap常见面试题解析

### HashMap

1. https://blog.csdn.net/weixin_41565013/article/details/93173607
2. 为什么hashMap的数组长度需要是2的n次幂？

   1. 使用2的n次幂作为数组大小，可以将很多运算简化为位运算，提高效率

      1. 使用 (length - 1) & hash确定数组位置
      2. 扩容时判断数据位置
3. 给定任意值，是如何将其扩展为2的N次幂的？
- 使用tableSizeFor方法
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1; // 先减一，防止其就是2的n次幂
    n |= n >>> 1; //通过或运算，将最左1之后所有位都填充为1
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    // 最后再加1，即可找到符合的最小的2的n次幂
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

2. hashMap的hash方法

   - 将key的hashcode与前16位进行异或运算，使前16位也参与到取余过程中，使hash更加均匀，减少hash碰撞

   - ```java
         static final int hash(Object key) {
             int h;
             return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
         }
     ```

3. resize（）中对链表的重新定位是怎么操作的？

   - 当数组长度为2的n次幂时，链表e的hashcode和旧数组长度oldCap满足以下关系： 
     - 如果(e.hash & oldCap) == 0，则扩容2倍后该node仍放在原数组下标位置
     - 如果(e.hash & oldCap) ！= 0，则扩容2倍后该node放在原数组下标+原数组长度位置

4. 链表转为红黑树的条件

   1. 链表长度大于8
   2. 数组长度（容量）大于64(否则只进行扩容而不转换为红黑树)

5. **红黑树的特性:**

   1. 每个节点或者是黑色，或者是红色。
   2.  根节点是黑色。
   3.  每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
   4.  如果一个节点是红色的，则它的子节点必须是黑色的。
   5.  从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。
   6. 红黑树新增/删除节点之后最多经过三次旋转即可达到平衡状态
   
6. HashMap线程安全方面会出现什么问题

   - 在jdk1.7中，在多线程环境下，扩容时会造成环形链或数据丢失。
   - 在jdk1.8中，在多线程环境下，会发生数据覆盖的情况（由头插法改为尾插法，避免了环形链的出现）

7. 已知hashMap存放数据量n的情况下，如何选择初始化hashMap大小？

   - 参考put时内部扩容 n / loadfactor + 1.0F
   
8. hash冲突的解决办法

   1. 链地址法，使用链表将hash冲突的节点放在同一个位置。hashMap的实现方式就是这个
   2. 开放定址法，如果hash冲突则使用fi(key) = (f(key)+di) MOD m (di=1,2,3,……,m-1) 重新计算位置。需要散列表足够大
   3. 再hash法，如果hash冲突则使用下一个hash函数重新计算
   4. 建立公共溢出区，将哈希表分为基本表和溢出表两部分，凡是和基本表发生冲突的元素，一律填入溢出表

   

### ConcurrentHashMap

1. get方法步骤

   1. 根据 key 计算出 hashcode 。
   2. 判断是否需要进行初始化。
   3. `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
   4. 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
   5. 如果都不满足，则利用 synchronized 锁写入数据。
   6. 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。

2. 关键属性

   - ```java
     // ConcurrentHashMap核心数组
         transient volatile Node<K,V>[] table;
      
         // 扩容时才会用的一个临时数组
         private transient volatile Node<K,V>[] nextTable;
         
         /**
          * table初始化和resize控制字段
          * 负数表示table正在初始化或resize。-1表示正在初始化，-N表示有N-1个线程正在resize操作
          * 当table为null的时候，保存初始化表的大小以用于创建时使用，或者直接采用默认值0
          * table初始化之后，保存下一次扩容的的大小，跟HashMap的threshold = loadFactor*capacity作用相同
          */
         private transient volatile int sizeCtl;
      
         // resize的时候下一个需要处理的元素下标为index=transferIndex-1
         private transient volatile int transferIndex;
      
         // 通过CAS无锁更新，ConcurrentHashMap元素总数，但不是准确值
         // 因为多个线程同时更新会导致部分线程更新失败，失败时会将元素数目变化存储在counterCells中
         private transient volatile long baseCount;
      
         // resize或者创建CounterCells时的一个标志位
         private transient volatile int cellsBusy;
      
         // 用于存储元素变动
         private transient volatile CounterCell[] counterCells;
     ```

3. 为什么concurrentHashMap在原resize的基础上加了& HASH_BITS？

   1. 为了屏蔽符号位

