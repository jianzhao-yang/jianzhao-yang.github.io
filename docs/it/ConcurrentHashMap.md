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