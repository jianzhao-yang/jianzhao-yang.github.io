## HashMap

精选博客: https://blog.csdn.net/weixin_41565013/article/details/93173607

## 结构

### 字段

```java
// -------------常量------------------
// 默认初始容量（数组默认初始化大小）
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30; //
// 默认的负载因子(在对map进行扩容时需要负载因子进行计算)
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//  桶的树化阈值：即 链表转成红黑树的阈值，在存储数据时，当链表长度 > 该值时，则将链表转换成红黑树
static final int TREEIFY_THRESHOLD = 8;
// 桶的链表还原阈值：即 红黑树转为链表的阈值，当在扩容（resize（））时（此时HashMap的数据存储位置会重新计算），在重新计算
// 存储位置后，当原有的红黑树内数量 < 6时，则将 红黑树转换成链表
static final int UNTREEIFY_THRESHOLD = 6;
// 最小树形化容量阈值：即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）
// 否则，若桶内元素太多时，则直接扩容，而不是树形化
//  为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
static final int MIN_TREEIFY_CAPACITY = 64;

// -----------------成员变量--------------
// 盛放元素/链表的数组(不能被序列化)
transient Node<K,V>[] table;
// 所有的键值对集合(不能被序列化) 只是一个访问table的迭代器,并没有实际保存值
transient Set<Map.Entry<K,V>> entrySet;
// map中键值对的个数(不能被序列化)
transient int size;
// 操作map的总的次数(不能被序列化)
transient int modCount;
// 下一个要调整大小的大小值（容量*负载系数）
int threshold;
// 哈希表的负载因子。
final float loadFactor; 

// --------------父类成员变量----------------
// 所有的键集合(不能被序列化) 只是一个访问table的迭代器,并没有实际保存值
transient Set<K>        keySet;
// 所有的值集合(不能被序列化) 只是一个访问table的迭代器,并没有实际保存值
transient Collection<V> values;
```

### 类

hashmap类大致有哪几部分组成?

1. 存储实际数据的table数组,负责其容量管理的size、threshold、loadFactor
2. 数组中的元素结构Node、Entry、TreeNode
3. 数据视图keySet、values、entrySet
4. 串行迭代器HashIterator、KeyIterator、ValueIterator和EntryIterator
5. 为Stream流服务的并发迭代器HashMapSpliterator、KeySpliterator、ValueSpliterator和EntrySetSpliterator
6. 记录map是否被并发修改过的modCount



## 扩容

1. 为什么hashMap的数组长度需要是2的n次幂？

   1. 使用2的n次幂作为数组大小，可以将很多运算简化为位运算，提高效率

      - 使用 (length - 1) & hash确定数组位置
      - 扩容时判断数据位置((hash & oldCap) == 0 则元素放在原数组槽arr[i],否则放在arr[i+oldCap])
2. 给定任意值，是如何将其扩展为2的N次幂的？
   - 使用tableSizeFor方法	

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1; // 先减一，防止其就是2的n次幂;另: 二进制左侧最大计数为肯定是1
    n |= n >>> 1; //通过或运算，将第二位也变成1,同样操作,将最左1之后所有位都填充为1
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

## 红黑树

2. 链表转为红黑树的条件

   1. 链表长度大于8
   2. 数组长度（容量）大于64(否则只进行扩容而不转换为红黑树)

3. **红黑树的特性:**

   1. 每个节点或者是黑色，或者是红色。
   2. 根节点是黑色。
   3. 每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
   4. 如果一个节点是红色的，则它的子节点必须是黑色的。
   5. 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。
   6. 红黑树新增/删除节点之后最多经过三次旋转即可达到平衡状态



## 亮点



## 问题

*HashMap在并发编程环境下有什么问题啊?*

- (1)多线程扩容，引起的死循环问题(jdk8已解决,由头插法改为尾插法，避免了环形链的出现) https://coolshell.cn/articles/9606.html
- (2)多线程put的时候可能导致元素丢失
- (3)put非null元素后get出来的却是null



2. 已知hashMap存放数据量n的情况下，如何选择初始化hashMap大小？

   - 参考put时内部扩容 n / loadfactor + 1.0F
3. hash冲突的解决办法

   1. 链地址法，使用链表将hash冲突的节点放在同一个位置。hashMap的实现方式就是这个
   2. 开放定址法，如果hash冲突则使用fi(key) = (f(key)+di) MOD m (di=1,2,3,……,m-1) 重新计算位置。需要散列表足够大
   3. 再hash法，如果hash冲突则使用下一个hash函数重新计算
   4. 建立公共溢出区，将哈希表分为基本表和溢出表两部分，凡是和基本表发生冲突的元素，一律填入溢出表
4. put方法流程

   1. 调用`key`的`hashCode`方法计算哈希值，并据此计算出数组下标index
   2. 如果发现当前的桶数组为`null`，则调用`resize()`方法进行初始化
   3. 如果没有发生哈希碰撞，则直接放到对应的桶中
   4. 如果发生哈希碰撞，且节点已经存在，就替换掉相应的`value`
   5. 如果发生哈希碰撞，且桶中存放的是树状结构，则挂载到树上
   6. 如果碰撞后为链表，添加到链表尾，如果链表长度超过`TREEIFY_THRESHOLD`默认是8，则将链表转换为树结构
   7. 数据`put`完成后，如果`HashMap`的总数超过`threshold`就要`resize` （如果数组大小<64，则只进行扩容）
5. 桶中的元素链表何时转换为红黑树，什么时候转回链表，为什么要这么设计？

   答：当同一个桶中的元素数量大于等于8的时候元素中的链表转换为红黑树，反之，当桶中的元素数量小于等于6的时候又会转为链表，这样做的原因是避免红黑树和链表之间频繁转换，引起性能损耗

11. `HashMap`如何处理`key`为`null`的键值对？

    答：放置在桶数组中下标为0的桶中

12. keySet、values、entrySet是什么?

    keySet、values、entrySet都只是指向table的迭代器,并没有保存额外的值,不占用实际空间.

    ps: 为什么通过idea调试可以看到这三个变量都有值? 因为idea展示的是变量的toString方法结果,而它们都重写了toString,在其中使用迭代器遍历汇总了结果进行显示

    以keySet为例：

    ```java
    // 返回一个新建或缓存的KeySet实例.(调用此方法之前keySet字段是null)
    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new KeySet();
            keySet = ks;
        }
        return ks;
    }
    // KeySet没有成员变量,只是一个迭代器
    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        // 看这里,迭代的时候生成一个KeyIterator
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
    // 也是一个包装,实际使用父类的nextNode()获取node
    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return super.nextNode().key; }
    }
    // 抽象一个顶级迭代器,这样keySet、valueSet和entrySet都可以用
    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot
        
    	// 因为是hashmap的内部类,所以可以访问外部类的成员变量
        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                // 找到第一个有数据的节点作为第一个next
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
    
        public final boolean hasNext() {
            return next != null;
        }
    	// 核心实现: 
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            // 如果next.next是空,且table不为空
            if ((next = (current = e).next) == null && (t = table) != null) {
                // 一直向后查找table,直到找到不为空的数组槽
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
    
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
    ```

13. 普通内部类特性(Outer和outer代表外部类和实例,Inner和inner代表内部类和实例)

    1. 内部类中可以访问外部类的成员变量
    2. 内部类可以使用outer.new Inner()创建属于该外部类的内部类实例
    3. 内部类中可以使用Outer.this.* 来访问外部类实例的成员变量
    4. 外部类继承内部类需要在构造方法中使用outer.super()来表明关联关系

14. 静态内部类特性

    1. 使用 new Outer.Inner()来实例化,不需要存在外部类实例
    2. 内部类无法访问外部类成员变量

15. 方法内部类

    1. 与普通内部类类似
    2. 只能是非static
    3. 使用范围只能在方法内

16. put方法

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)// 如果是空数组
            n = (tab = resize()).length;// 扩容并设置数组长度
        if ((p = tab[i = (n - 1) & hash]) == null)// 如果key所在数组槽位为空
            tab[i] = newNode(hash, key, value, null);// 直接构造新node放入
        else {// 槽位已经有值
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))// 如果key就是数组槽位key
                e = p;
            else if (p instanceof TreeNode)// 如果该槽位中是红黑树
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);// 树中插入
            else {// 链表插入
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {// 如果未找到
                        p.next = newNode(hash, key, value, null);// 链表插入
                        if (binCount >= TREEIFY_THRESHOLD - 1) // 如果需要扩容
                            treeifyBin(tab, hash);// 链表转红黑树
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))// 如果已存在,不插入
                        break;
                    p = e;// 既不是已存在又不是最后一个,则继续
                }
            }
            if (e != null) { // 如果key已存在
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)// 如果允许覆盖旧值
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```

17. hashmap和redis的哈希表有什么区别?

    1. Redis 字典使用的时哈希表作为底层，并且每个字典维护了**两个哈希表**，ht[0] 时主要使用的哈希表，而ht[1] 是在rehash过程是才会使用到的表。
    2. 哈希表的底层同样是使用了数组 + 链表的结构， 与Java 中HashMap 相似，只不过Java8 以后增加了**红黑树**，在特定情况下会替换链表。
    3. 哈希表增加元素遇到哈希冲突是会将新添加的元素放到**链表头**，而Java HashMap会将其放到链表尾，
    4. 扩容过程中redis的字典是**渐进式扩容**，扩容期间还是可以进行操作的，而Java的HashMap扩容需要一次性完成。

    

18. - 

