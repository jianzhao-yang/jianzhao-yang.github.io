# 计算机网络

## TCP

### 基础

为什么需要TCP? 

> 

## socket编程



### socket结构

```c++
struct socket {
    struct file     *file;// Linux一切皆文件,所以文件和socket双向都有指针
    struct sock     *sk;// 要发送/接收的网络数据都在这里
    struct proto_ops *ops;// 所有操作方法的集合
}
struct sock{
    struct sk_buff_head sk_receive_queue;// 接收队列,接收到的网络数据会由网卡DMA到这里
    struct socket_wq __rcu	*sk_wq;// 等待队列,线程在这里阻塞等待数据.(它是一个复杂的结构)
}
```

## epoll

### epoll结构

```c
// file：fs/eventpoll.c
struct eventpoll {
    //sys_epoll_wait用到的等待队列;
    struct wait_queue_head_t wq;
    //接收就绪的描述符都会放到这里
    struct list_head rdllist;
    //每个epoll对象中都有一颗红黑树,红黑树里放的是epitem
    struct rb_root rbr;
}
```

eventpoll 这个结构体中的几个成员的含义如下：

- **wq：** 等待队列链表。软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。等待队列里的对象结构: 被阻塞的线程current;回调函数callback;等待队列唤醒方式: callback(current,args);
- **rbr：** 一棵红黑树。为了支持对海量连接的高效查找、插入和删除，eventpoll 内部使用了一棵红黑树。通过这棵树来管理用户进程下添加进来的所有 socket 连接。
- **rdllist：** 就绪的描述符的链表。当有的连接就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。

```c
//file: fs/eventpoll.c
struct epitem {
    //红黑树节点(和红黑树互指)
    struct rb_node rbn;
    // epoll的就绪列表rdllist放的就是这个数据集合
    struct list_head rdllink;
    //socket文件描述符信息(可以理解为指向socket)
    struct epoll_filefd ffd;
    //所归属的 eventpoll 对象
    struct eventpoll *ep;
    //等待队列
    struct list_head pwqlist;
}
```

### epoll和socket关系

如下图,可知:

1. socket无法感知到epoll的存在,只有通过等待队列中的回调函数才能与epoll交互;
2. epoll中红黑树上的节点epitem持有指向socket描述符的引用,所以可以直接访问socket;
3. 使用epoll后,socket的等待队列上只有一个callback函数,线程不再阻塞;阻塞位置改为epoll上面的等待队列wq

![](https://raw.githubusercontent.com/jianzhao-yang/picbed/main/pic/epoll%E5%92%8Csocket%E5%85%B3%E7%B3%BB.png)

socket的等待队列sk_wq中注册的回调函数ep_poll_callback

```c
//file: fs/eventpoll.c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    //获取 wait 对应的 epitem
    struct epitem *epi = ep_item_from_wait(wait);

    //获取 epitem 对应的 eventpoll 结构体
    struct eventpoll *ep = epi->ep;

    //1. 将当前epitem 添加到 eventpoll 的就绪队列中
    list_add_tail(&epi->rdllink, &ep->rdllist);

    //2. 查看 eventpoll 的等待队列上是否有在等待
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq); // 调用epoll_wait()函数调用时注册的函数default_wake_function (),该函数尝试唤醒阻塞的线程,并执行相关操作
}
```

### epoll执行流程

![epoll执行流程](C:\Users\yangjc\Pictures\技术\epoll执行流程.jpg)
