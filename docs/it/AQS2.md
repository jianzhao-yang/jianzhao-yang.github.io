关键字段

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
    	// 核心字段,持有锁的线程
        private Thread exclusiveOwnerThread;
}
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements Serializable {
    // 核心字段,指示clh队列的头部
    private volatile Node head;
    // 核心字段,指示clh队列的尾部
    private volatile Node tail;
    // 核心字段,指示队列的当前状态.
    private volatile int state;
    // 节点,代表等待获取锁的线程
    static final class Node {
        // 核心字段,当前节点等待状态: 1:已取消,-1:被唤醒的当前节点,-2:条件节点,3:广播节点
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        // 当前节点代表的线程
        volatile Thread thread;
        Node nextWaiter;
    }
    // 条件节点
    public class ConditionObject implements Condition, Serializable {
        private Node firstWaiter;
        private Node lastWaiter;
        private static final int REINTERRUPT = 1;
        private static final int THROW_IE = -1;
    }
}
```

关键方法

```java
// 获取n个资源
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)){
        // 响应中断
        selfInterrupt();
    }
}
// ReentrantLock.NonfairSync#nonfairTryAcquire的一种实现
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {// 如果当前状态为0,代表资源空闲,则直接获取
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {// 如果当前已经获取锁,则执行可重入的方法
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
// 加入等待队列
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 尝试直接加入队尾
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 如果加入失败,说明存在竞争,则自旋尝试加入队尾
    enq(node);
    return node;
}
// 在队列中等待获取锁
final boolean acquireQueued(final Node node, int arg) {
     boolean failed = true;//标记是否成功拿到资源
     try {
         boolean interrupted = false;//标记等待过程中是否被中断过
         //又是一个“自旋”！
         for (;;) {
             final Node p = node.predecessor();//拿到前驱
             //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源
             //（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
             if (p == head && tryAcquire(arg)) {
                 //拿到资源后，将head指向该结点。
                 //所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                 setHead(node);
                 // setHead中node.prev已置为null，此处再将head.next置为null，
                 // 就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                 p.next = null; 
                 failed = false; // 成功获取资源
                 return interrupted;//返回等待过程中是否被中断过
             }
             //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。
             //如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
             if (shouldParkAfterFailedAcquire(p, node) &&
                 parkAndCheckInterrupt())
                 interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
         }
     } finally {
         // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），
         // 那么取消结点在队列中的等待。
         if (failed) 
             cancelAcquire(node);
     }
 }
```