# Netty

在了解netty之前,必须先搞懂C语言socket编程和Linux epoll事件模型,否则会一直是一头雾水;

https://github.com/yanfeizhang/coder-kung-fu

# 概念

https://mp.weixin.qq.com/s/DS52g3bibU9kNH75UFxGqA

```java
public final class EchoServer {
 public static void main(String[] args) throws Exception {
     EventLoopGroup bossGroup = new NioEventLoopGroup(1);// 只用来处理accept()
     EventLoopGroup workerGroup = new NioEventLoopGroup();// 处理read(),write()及send(),recv()等
     final EchoServerHandler serverHandler = new EchoServerHandler();

     ServerBootstrap b = new ServerBootstrap();
     b.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .option(ChannelOption.SO_BACKLOG, 100)
         .handler(new LoggingHandler(LogLevel.INFO))
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 ChannelPipeline p = ch.pipeline();
                 if (sslCtx != null) {
                     p.addLast(sslCtx.newHandler(ch.alloc()));
                 }
                 p.addLast(serverHandler);
             }
         });

     // bind(PORT)真正打开socket,开始bind()和listen()
     ChannelFuture f = b.bind(PORT).sync();
     ......
 }
}

```



## NioEventLoopGroup

NioEventLoopGroup简化

可以看到NioEventLoopGroup其实就是一个包装后的线程池;

两个线程池:

1. 主线程池只做socket编程流程中的accept步骤,即接受客户端连接;
2. 从线程池做剩余的读、写等操作；
3. 注意，它们只负责将客户端的连接和数据映射到本机内核，具体业务处理不是它们负责

```java
public class NioEventLoopGroup {
    private final EventExecutor[] children;// 实际是NioEventLoop子类
}
```

## NioEventLoop

NioEventLoop是对整个多路复用操作的封装;

```java
public class NioEventLoop extend EventExecutor{
    private Selector selector; // epoll等多路复用实现的封装
    private SelectedSelectionKeySet selectedKeys;//待处理已就绪事件
    private final Queue<Runnable> taskQueue;// 待处理任务
    private volatile Thread thread;// 线程
}
```



## Selector

因为在不同平台,多路复用的实现不同,所以不能用一个统一的Selector实现类来代表,所以需要使用SelectorProvider来进行创建;

selector就是epoll的封装;

```java
//file:java/nio/channels/spi/SelectorProvider.java
public abstract class SelectorProvider {

    public static SelectorProvider provider() {
     // 1. java.nio.channels.spi.SelectorProvider 属性指定实现类
        // 2. SPI 指定实现类
        ......

        // 3. 默认实现，Windows 和 Linux 下不同
        provider = sun.nio.ch.DefaultSelectorProvider.create();
        return provider;
    }
}
// 注意: 此类在Linux和windows下不同
//file:sun/nio/ch/DefaultSelectorProvider.java
public class DefaultSelectorProvider {
 public static SelectorProvider create() {
        String osname = AccessController
            .doPrivileged(new GetPropertyAction("os.name"));
        if (osname.equals("Linux"))
            return createProvider("sun.nio.ch.EPollSelectorProvider");
    }

}
```

## Channel

channel是对socket的封装;

```java
public class NioServerSocketChannel{
	private final ServerSocket socket;// 注意这里只是一个示例,实际上不是包含而是重新实现
    private final DefaultChannelPipeline pipeline;// pipeline保存了使用者注册的在socket各个阶段进行的操作
}
```



# netty为何高性能

https://blog.csdn.net/qqq3117004957/article/details/106435076

- **非阻塞 IO**

  Netty 采用了 IO 多路复用技术，让多个 IO 的阻塞复用到一个 select 线程阻塞上，能够有效的应对大量的并发请求

- **高效的 Reactor 线程模型**

  Netty 服务端采用 Reactor 主从多线程模型

  1. 主线程：Acceptor 线程池用于监听 Client 的 TCP 连接请求
  2. 从线程：Client 的 IO 操作都由一个特定的 NIO 线程池负责，负责消息的读取、解码、编码和发送
  3. Client 连接有很多，但是 NIO 线程数是比较少的，一个 NIO 线程可以同时绑定到多个 Client，同时一个 Client 只能对应一个线程，避免出现线程安全问题

- **无锁化串行设计**

  串行设计：消息的处理尽可能在一个线程内完成，期间不进行线程切换，避免了多线程竞争和同步锁的使用

- **高效的并发编程**

  Netty 的高效并发编程主要体现在如下几点

  1. volatile 的大量、正确使用
  2. CAS 和原子类的广泛使用
  3. 线程安全容器的使用
  4. 通过读写锁提升并发性能

- **高性能的序列化框架**

  Netty 默认提供了对 Google Protobuf 的支持，通过扩展 Netty 的编解码接口，可以实现其它的高性能序列化框架

- **零拷贝**

  1. **Netty 的接收和发送 ByteBuffer 采用 DirectByteBuffer，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝**。如果使用传统的堆内存（HeapByteBuffer）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。**相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝**
  2. Netty 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那样方便的对组合 Buffer 进行操作，避免了传统通过内存拷贝的方式将几个小 Buffer 合并成一个大的 Buffer。
  3. **Netty 的文件传输采用了 transferTo() 方法，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write() 方式导致的内存拷贝问题。**

- **内存池**

  基于对象池的 ByteBuf 可以重用 ByteBuf 对象，内部维护了一个内存池，可以循环利用已创建的 ByteBuf，提升内存的使用效率，降低由于高负载导致的频繁 GC。测试表明使用内存池后的 Nety 在高负载、大并发的冲击下内存和 GC 更加平稳

- **灵活的 TCP 参数配置能力**

  合理设置 TCP 参数在某些场景下对于性能的提升可以起到显著的效果，例如 SO_RCVBUF 和 SO_SNDBUF。如果设置不当，对性能的影响是非常大的

  1. SO_RCVBUF 和 SO_SNDBUF：通常建议值为 128K 或者 256K；
  2. SO_TCPNODELAY：NAGLE 算法通过将缓冲区内的小封包自动相连，组成较大的封包，阻止大量小封包的发送阻塞网络，从而提高网络应用效率。但是对于时延敏感的应用场景需要关闭该优化算法；
  3. 软中断：如果 Linux 内核版本支持 RPS（2.6.35 以上版本），开启 RPS 后可以实现软中断，提升网络吞吐量。RPS 根据数据包的源地址，目的地址以及目的和源端口，计算出一个 hash 值，然后根据这个 hash 值来选择软中断运行的 cpu，从上层来看，也就是说将每个连接和 cpu 绑定，并通过这个 hash 值，来均衡软中断在多个 cpu 上，提升网络并行处理性能

# Reactor模型

1. Netty 抽象出两组线程池 BossGroup 专门负责接收客户端的连接， Worker Group 专门负责网络的读写

2. Boss Group 和 Worker Group 类型都是 NioEventLoopGroup

3. NioEventLoop Group 相当于一个事件循环组，这个组中含有多个事件循环，每一个事件循环是 NioEventLoop

4. NioEventLoop 表示一个不断循环的执行处理任务的线程，每个 NioEventLoop 都有一个 selector, 用于监听绑定在其上的 socket 的网络通讯

5. NioEventLoop Group 可以有多个线程，即可以含有多个 NioEventLoop

6. 每个 Boss NioEventLoop 循环执行的步骤有 3 步

  1. 轮询 accept 事件

  2. 处理 accept 事件，与 client 建立连接，生成 NioScocketchannel, 并将其注册到某个 worker NIOEventLoop 上的 selector

  3. 处理任务队列的任务，即 runAllTasks

7. 每个 Worker NIOEventLoop 循环执行的步骤

   1. 轮询 read, write 事件

   2. 处理 0 事件，即 ead,wite 事件，在对应 NioScocketChannel 处理

   3. 处理任务队列的任务，即 runAlllask

8. 每个 Worker NIOE ventloop 处理业务时，会使用 pipeline 管道， pipeline 中包含了 channel, 即通过 pipeline 可以获取到对应通道，管道中维护了很多的处理器

#  I/O 多路复用之 select、poll、epoll 

https://www.cnblogs.com/aspirant/p/9166944.html

https://www.cnblogs.com/liugp/p/10962047.html

## select

### 优点

- 良好的跨平台性
- 延时低

### 缺点

- 每次调用select，都需要把fd集合从用户态**拷贝**到内核态，这个开销在fd很多时会很大
- 同时每次调用select都需要在内核**遍历**传递进来的所有fd，这个开销在fd很多时也很大
- select支持的文件描述符数量太小了，默认是1024

```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分 3 类，分别是 writefds、readfds、和 exceptfds。调用后 select 函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有 except），或者超时（timeout 指定等待时间，如果立即返回设为 null 即可），函数返回。当 select 函数返回后，可以 通过遍历 fdset，来找到就绪的描述符。

select 目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select 的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在 Linux 上一般为 1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但 是这样也会造成效率的降低。



## poll

### 优点

- 使用链表储存,去除了最大文件描述符限制
- 其它跟select无区别

```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```



不同与 select 使用三个位图来表示三个 fdset 的方式，poll 使用一个 pollfd 的指针实现。

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

pollfd 结构包含了要监视的 event 和发生的 event，不再使用 select“参数 - 值” 传递的方式。同时，pollfd 并没有最大数量限制（但是数量过大后性能也是会下降）。 和 select 函数一样，poll 返回后，需要轮询 pollfd 来获取就绪的描述符。

从上面看，select 和 poll 都需要在返回后，`通过遍历文件描述符来获取已经就绪的socket`。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。



## epoll

### 优点

-  没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）；
- 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；
- Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。
- 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。

### 缺点

- epoll是Linux系统独有的

epoll 是在 2.6 内核中提出的，是之前的 select 和 poll 的增强版本。相对于 select 和 poll 来说，epoll 更加灵活，没有描述符限制。epoll 使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的 copy 只需一次。

### 一 epoll 操作过程



epoll 操作过程需要三个接口，分别如下：



```
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```



**1. int epoll_create(int size);**
创建一个 epoll 的句柄，size 用来告诉内核这个监听的数目一共有多大，这个参数不同于 select() 中的第一个参数，给出最大监听的 fd+1 的值，`参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议`。
当创建好 epoll 句柄后，它就会占用一个 fd 值，在 linux 下如果查看 / proc / 进程 id/fd/，是能够看到这个 fd 的，所以在使用完 epoll 后，必须调用 close() 关闭，否则可能导致 fd 被耗尽。



**2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；**
函数是对指定描述符 fd 执行 op 操作。
- epfd：是 epoll_create() 的返回值。
- op：表示 op 操作，用三个宏来表示：添加 EPOLL_CTL_ADD，删除 EPOLL_CTL_DEL，修改 EPOLL_CTL_MOD。分别添加、删除和修改对 fd 的监听事件。
- fd：是需要监听的 fd（文件描述符）
- epoll_event：是告诉内核需要监听什么事，struct epoll_event 结构如下：



```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```



**3. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);**
等待 epfd 上的 io 事件，最多返回 maxevents 个事件。
参数 events 用来从内核得到事件的集合，maxevents 告之内核这个 events 有多大，这个 maxevents 的值不能大于创建 epoll_create() 时的 size，参数 timeout 是超时时间（毫秒，0 会立即返回，-1 将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回 0 表示已超时。



### 二 工作模式

　epoll 对文件描述符的操作有两种模式：**LT（level trigger）**和 **ET（edge trigger）**。LT 模式是默认模式，LT 模式与 ET 模式的区别如下：
　　**LT 模式**：当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，`应用程序可以不立即处理该事件`。下次调用 epoll_wait 时，会再次响应应用程序并通知此事件。
　　**ET 模式**：当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，`应用程序必须立即处理该事件`。如果不处理，下次调用 epoll_wait 时，不会再次响应应用程序并通知此事件。

#### 1. LT 模式

LT(level triggered) 是缺省的工作方式，并且同时支持 block 和 no-block socket. 在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 IO 操作。如果你不作任何操作，内核还是会继续通知你的。

#### 2. ET 模式

ET(edge-triggered) 是高速工作方式，只支持 no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过 epoll 告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了 (比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误）。但是请注意，如果一直不对这个 fd 作 IO 操作 (从而导致它再次变成未就绪)，内核不会发送更多的通知 (only once)

ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读 / 阻塞写操作把处理多个文件描述符的任务饿死。

#### 3. 总结

**假如有这样一个例子：**
1. 我们已经把一个用来从管道中读取数据的文件句柄 (RFD) 添加到 epoll 描述符
2. 这个时候从管道的另一端被写入了 2KB 的数据
3. 调用 epoll_wait(2)，并且它会返回 RFD，说明它已经准备好读取操作
4. 然后我们读取了 1KB 的数据
5. 调用 epoll_wait(2)......

**LT 模式：**
如果是 LT 模式，那么在第 5 步调用 epoll_wait(2) 之后，仍然能受到通知。

**ET 模式：**
如果我们在第 1 步将 RFD 添加到 epoll 描述符的时候使用了 EPOLLET 标志，那么在第 5 步调用 epoll_wait(2) 之后将有可能会挂起，因为剩余的数据还存在于文件的输入缓冲区内，而且数据发出端还在等待一个针对已经发出数据的反馈信息。只有在监视的文件句柄上发生了某个事件的时候 ET 工作模式才会汇报事件。因此在第 5 步的时候，调用者可能会放弃等待仍在存在于文件输入缓冲区内的剩余数据。

当使用 epoll 的 ET 模型来工作时，当产生了一个 EPOLLIN 事件后，
读数据的时候需要考虑的是当 recv() 返回的大小如果等于请求的大小，那么很有可能是缓冲区还有数据未读完，也意味着该次事件还没有处理完，所以还需要再次读取：



```
while(rs){
  buflen = recv(activeevents[i].data.fd, buf, sizeof(buf), 0);
  if(buflen < 0){
    // 由于是非阻塞的模式,所以当errno为EAGAIN时,表示当前缓冲区已无数据可读
    // 在这里就当作是该次事件已处理处.
    if(errno == EAGAIN){
        break;
    }
    else{
        return;
    }
  }
  else if(buflen == 0){
     // 这里表示对端的socket已正常关闭.
  }

 if(buflen == sizeof(buf){
      rs = 1;   // 需要再次读取
 }
 else{
      rs = 0;
 }
}
```



**Linux 中的 EAGAIN 含义**

Linux 环境下开发经常会碰到很多错误 (设置 errno)，其中 EAGAIN 是其中比较常见的一个错误 (比如用在非阻塞操作中)。
从字面上来看，是提示再试一次。这个错误经常出现在当应用程序进行一些非阻塞 (non-blocking) 操作 (对文件或 socket) 的时候。

例如，以 O_NONBLOCK 的标志打开文件 / socket/FIFO，如果你连续做 read 操作而没有数据可读。此时程序不会阻塞起来等待数据准备就绪返回，read 函数会返回一个错误 EAGAIN，提示你的应用程序现在没有数据可读请稍后再试。
又例如，当一个系统调用 (比如 fork) 因为没有足够的资源 (比如虚拟内存) 而执行失败，返回 EAGAIN 提示其再调用一次(也许下次就能成功)。



### epoll 总结

在 select/poll 中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而 **epoll 事先通过 epoll_ctl() 来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似 callback 的回调机制，迅速激活这个文件描述符，当进程调用 epoll_wait() 时便得到通知**。(`此处去掉了遍历文件描述符，而是通过监听回调的的机制`。这正是 epoll 的魅力所在。)

**epoll 的优点主要是一下几个方面：**
1. **监视的描述符数量不受限制**，它所支持的 FD 上限是最大可以打开文件的数目，这个数字一般远大于 2048, 举个例子, 在 1GB 内存的机器上大约是 10 万左 右，具体数目可以 cat /proc/sys/fs/file-max 察看, 一般来说这个数目和系统内存关系很大。select 的最大缺点就是进程打开的 fd 是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案 (Apache 就是这样实现的)，不过虽然 linux 上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。

1. **IO 的效率不会随着监视 fd 的数量的增长而下降**。epoll 不同于 select 和 poll 轮询的方式，而是通过每个 fd 定义的回调函数来实现的。只有就绪的 fd 才会执行回调函数。

**如果没有大量的 idle -connection 或者 dead-connection，epoll 的效率并不会比 select/poll 高很多，但是当遇到大量的 idle- connection，就会发现 epoll 的效率大大高于 select/poll。**

# 零拷贝

零拷贝的应用程序要求内核（kernel）直接将数据从磁盘文件拷贝到套接字（Socket），而**无须通过应用程序**。零拷贝不仅提高了应用程序的性能，而且减少了内核和用户模式见上下文切换。 

## 数据传输：传统方法

从文件中读取数据，并将数据传输到网络上的另一个程序的场景：从下图可以看出，拷贝的操作需要 4 次用户模式和内核模式之间的上下文切换，而且在操作完成前数据被复制了 **4 次**。

### 数据被拷贝了4次

![img](https://images2018.cnblogs.com/blog/167213/201806/167213-20180626182656662-1723722762.png)

### 上下文切换了4次

![img](https://images2018.cnblogs.com/blog/167213/201806/167213-20180626183033495-963753033.png)





## ![img](https://images2018.cnblogs.com/blog/167213/201807/167213-20180703153135997-476874108.png)



从磁盘中 copy 放到一个内存 buf 中，然后将 buf 通过 socket 传输给用户, 下面是伪代码实现：

read(file, tmp_buf, len);
write(socket, tmp_buf, len);

从图中可以看出文件经历了 4 次 copy 过程：

1. 首先，调用 read 方法，文件从 user 模式拷贝到了 kernel 模式；（用户模式 -> 内核模式的上下文切换，在内部发送 sys_read() 从文件中读取数据，存储到一个内核地址空间缓存区中）

2. 之后 CPU 控制将 kernel 模式数据拷贝到 user 模式下；（内核模式 -> 用户模式的上下文切换，read() 调用返回，数据被存储到用户地址空间的缓存区中）

3. 调用 write 时候，先将 user 模式下的内容 copy 到 kernel 模式下的 socket 的 buffer 中（用户模式 -> 内核模式，数据再次被放置在内核缓存区中，send（）套接字调用）

4. 最后将 kernel 模式下的 socket buffer 的数据 copy 到网卡设备中；（send 套接字调用返回）

从图中看 2，3 两次 copy 是多余的，数据从 kernel 模式到 user 模式走了一圈，浪费了 2 次 copy。



## 数据传输：零拷贝方法

从传统的场景看，会注意到上图，第 2 次和第 3 次拷贝根本就是多余的。应用程序只是起到缓存数据被将传回到套接字的作用而已，别无他用。

应用程序使用 zero-copy 来请求 kernel 直接把 disk 的数据传输到 socket 中，而不是通过应用程序传输。zero-copy 大大提高了应用程序的性能，并且减少了 kernel 和 user 模式的上下文切换。

### Linux TransferTo

数据可以直接从 read buffer 读缓存区传输到套接字缓冲区，也就是省去了将操作系统的 read buffer 拷贝到程序的 buffer，以及从程序 Tbuffer 拷贝到 socket buffer 的步骤，直接将 read buffer 拷贝到 socket buffer。JDK NIO 中的的`transferTo()` 方法就能够让您实现这个操作，这个实现依赖于操作系统底层的 sendFile（）实现的：

```
public void transferTo(long position, long count, WritableByteChannel target);
```

底层调用 sendFile 方法：

```
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```



![img](https://images2018.cnblogs.com/blog/167213/201807/167213-20180703160728436-527898617.png)





![img](https://images2018.cnblogs.com/blog/167213/201807/167213-20180703160759059-335886092.png)





使用了 zero-copy 技术后，整个过程如下：

1. transferTo() 方法使得文件的内容直接 copy 到了一个 read buffer（kernel buffer）中

2. 然后数据（kernel buffer）copy 到 socket buffer 中

3. 最后将 socket buffer 中的数据 copy 到网卡设备（protocol engine）中传输；

这个显然是一个伟大的进步：**这里上下文切换从 4 次减少到 2 次，同时把数据 copy 的次数从 4 次降低到 3 次**；

**但是这是 zero-copy 么，答案是否定的；**



### Linux Sendfile 

#### Linux2.4

Linux2.4 内核对 sendfile 做了改进，如图：

![img](https://images2018.cnblogs.com/blog/167213/201807/167213-20180703165126930-217245674.png)

改进后的处理过程如下：

1. 将文件拷贝到 kernel buffer 中；(DMA 引擎将文件内容 copy 到内核缓存区)
2. 向 socket buffer 中追加当前要发生的数据在 kernel buffer 中的**位置和偏移量**；
3. 根据 socket buffer 中的位置和偏移量直接将 kernel buffer 的数据 copy 到网卡设备（protocol engine）中；

从图中看到，linux 2.1 内核中的 “**数据被 copy 到 socket buffe**r” 的动作，在 Linux2.4 内核做了优化，取而代之的是只包含关于数据的位置和长度的信息的描述符被追加到了 socket buffer 缓冲区中。**DMA 引擎直接把数据从内核缓冲区传输到协议引擎**（protocol engine），从而消除了最后一次 CPU copy。经过上述过程，数据只经过了 2 次 copy 就从磁盘传送出去了。这个才是真正的 Zero-Copy(这里的零拷贝是针对 kernel 来讲的，数据在 kernel 模式下是 Zero-Copy)。

### Java transferTo()

**底层使用的是Linux的sendfile**

正是 Linux2.4 的内核做了改进，Java 中的 TransferTo() 实现了 Zero-Copy, 如下图：

![img](https://mmbiz.qpic.cn/mmbiz_png/HV4yTI6PjbKgic1MvAjpofibyQwiauK9swkIr4RdZunnVfczbFdpJT1bFQRBdicslA4bW92FJBvtiacojo6KgBDial6A/640)



Zero-Copy 技术的使用场景有很多，比如 Kafka, 又或者是 Netty 等，可以大大提升程序的性能。