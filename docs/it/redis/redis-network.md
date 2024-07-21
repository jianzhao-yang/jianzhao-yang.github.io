# redis网络模型

> 我们都知道 redisIO 模型 使用的是 IO 多路复用 ，我们也都知道它很高效，但是他为什么高效呢？redis 为何会选择这种方式？

[](#用户空间和内核态空间 "用户空间和内核态空间")用户空间和内核态空间

[](#为什么要区分用户和内核 "为什么要区分用户和内核")为什么要区分用户和内核

 计算机硬件包括，如 cpu，内存，网卡等等，内核（通过寻址空间）可以操作硬件的，但是内核需要不同设备的驱动，有了这些驱动之后，内核就可以去对计算机硬件去进行 内存管理，文件系统的管理，进程的管理等等

[![](http://blog.czk.pub/assets/1653896065386.png)](http://blog.czk.pub/assets/1653896065386.png)

 我们想要用户的应用来访问，计算机就必须要通过对外暴露的一些接口，才能访问到，从而简介的实现对内核的操控，但是内核本身上来说也是一个应用，所以他本身也需要一些内存，cpu 等设备资源，用户应用本身也在消耗这些资源，如果不加任何限制，用户去操作随意的去操作我们的资源，就有可能导致一些冲突，甚至有可能导致我们的系统出现无法运行的问题，因此我们需要**把用户和内核隔离开**

### [](#进程寻址空间 "进程寻址空间")进程寻址空间

> 进程的寻址空间划分成两部分：**内核空间、用户空间**

 什么是寻址空间呢？我们的应用程序也好，还是内核空间也好，都是没有办法直接去物理内存的，而是通过分配一些虚拟内存映射到物理内存中，我们的内核和应用程序去访问虚拟内存的时候，就需要一个虚拟地址，这个地址是一个无符号的整数。

 比如一个 32 位的操作系统，他的带宽就是 32，他的虚拟地址就是 2 的 32 次方，也就是说他寻址的范围就是 0~2 的 32 次方， 这片寻址空间对应的就是 2 的 32 个字节，就是 4GB，这个 4GB，会有 3 个 GB 分给用户空间，会有 1GB 给内核系统

[![](http://blog.czk.pub/assets/1653896377259.png)](http://blog.czk.pub/assets/1653896377259.png)

 在 linux 中，他们权限分成两个等级，0 和 3，用户空间只能执行受限的命令（Ring3），而且不能直接调用系统资源，必须通过内核提供的接口来访问内核空间可以执行特权命令（Ring0），调用一切系统资源，所以一般情况下，用户的操作是运行在用户空间，而内核运行的数据是在内核空间的，而有的情况下，一个应用程序需要去调用一些特权资源，去调用一些内核空间的操作，所以此时他俩需要在用户态和内核态之间进行切换。

比如：

**Linux 系统为了提高 IO 效率，会在用户空间和内核空间都加入缓冲区：**

*   **写数据时，要把用户缓冲数据拷贝到内核缓冲区，然后写入设备**
*   **读数据时，要从设备读取数据到内核缓冲区，然后拷贝到用户缓冲区**

 针对这个操作：我们的用户在写读数据时，会去向内核态申请，想要读取内核的数据，而内核数据要去等待驱动程序从硬件上读取数据，当从磁盘上加载到数据之后，内核会将数据写入到内核的缓冲区中，然后再将数据拷贝到用户态的 buffer 中，然后再返回给应用程序，整体而言，速度慢，就是这个原因，为了加速，我们希望 read 也好，还是 wait for data 也最好都不要等待，或者时间尽量的短。

[![](http://blog.czk.pub/assets/1653896687354.png)](http://blog.czk.pub/assets/1653896687354.png)

[](#网络模型 "网络模型")网络模型
--------------------

### [](#阻塞IO "阻塞IO")阻塞 IO

*   过程 1：应用程序想要去读取数据，他是无法直接去读取磁盘数据的，他需要先到内核里边去等待内核操作硬件拿到数据，这个过程是需要等待的，等到内核从磁盘上把数据加载出来之后，再把这个数据写给用户的缓存区。
*   过程 2：如果是阻塞 IO，那么整个过程中，用户从发起读请求开始，一直到读取到数据，都是一个阻塞状态。

[![](http://blog.czk.pub/assets/1653897115346.png)](http://blog.czk.pub/assets/1653897115346.png)

用户去读取数据时，会去先发起 recvform 一个命令，去尝试从内核上加载数据，如果内核没有数据，那么用户就会等待，此时内核会去从硬件上读取数据，内核读取数据之后，会把数据拷贝到用户态，并且返回 ok，整个过程，都是阻塞等待的，这就是阻塞 IO

总结如下：

顾名思义，阻塞 IO 就是两个阶段都必须阻塞等待：

**阶段一：**

*   用户进程尝试读取数据（比如网卡数据）
*   此时数据尚未到达，内核需要等待数据
*   此时用户进程也处于阻塞状态

阶段二：

*   数据到达并拷贝到内核缓冲区，代表已就绪
*   将内核数据拷贝到用户缓冲区
*   拷贝过程中，用户进程依然阻塞等待
*   拷贝完成，用户进程解除阻塞，处理数据

可以看到，阻塞 IO 模型中，用户进程在两个阶段都是阻塞状态。

[![](http://blog.czk.pub/assets/1653897270074.png)](http://blog.czk.pub/assets/1653897270074.png)

### [](#非阻塞IO "非阻塞IO")非阻塞 IO

顾名思义，非阻塞 IO 的 recvfrom 操作会立即返回结果而不是阻塞用户进程。

阶段一：

*   用户进程尝试读取数据（比如网卡数据）
*   此时数据尚未到达，内核需要等待数据
*   返回异常给用户进程
*   用户进程拿到 error 后，再次尝试读取
*   循环往复，直到数据就绪

阶段二：

*   将内核数据拷贝到用户缓冲区
*   拷贝过程中，用户进程依然阻塞等待
*   拷贝完成，用户进程解除阻塞，处理数据
*   可以看到，非阻塞 IO 模型中，用户进程在第一个阶段是非阻塞，第二个阶段是阻塞状态。虽然是非阻塞，但性能并没有得到提高。而且忙等机制会导致 CPU 空转，CPU 使用率暴增。

[![](http://blog.czk.pub/assets/1653897490116.png)](http://blog.czk.pub/assets/1653897490116.png)

### [](#信号驱动 "信号驱动")信号驱动

信号驱动 IO 是与内核建立 SIGIO 的信号关联并设置回调，当内核有 FD 就绪时，会发出 SIGIO 信号通知用户，期间用户应用可以执行其它业务，无需阻塞等待。

阶段一：

*   用户进程调用 sigaction ，注册信号处理函数
*   内核返回成功，开始监听 FD
*   用户进程不阻塞等待，可以执行其它业务
*   当内核数据就绪后，回调用户进程的 SIGIO 处理函数

阶段二：

*   收到 SIGIO 回调信号
*   调用 recvfrom ，读取
*   内核将数据拷贝到用户空间
*   用户进程处理数据

[![](http://blog.czk.pub/assets/1653911776583.png)](http://blog.czk.pub/assets/1653911776583.png)

当有大量 IO 操作时，信号较多，SIGIO 处理函数不能及时处理可能导致信号队列溢出，而且内核空间与用户空间的频繁信号交互性能也较低。

### [](#异步IO "异步IO")异步 IO

这种方式，不仅仅是用户态在试图读取数据后，不阻塞，而且当内核的数据准备完成后，也不会阻塞

他会由内核将所有数据处理完成后，由内核将数据写入到用户态中，然后才算完成，所以性能极高，不会有任何阻塞，全部都由内核完成，可以看到，异步 IO 模型中，用户进程在两个阶段都是非阻塞状态。

[![](http://blog.czk.pub/assets/1653911877542.png)](http://blog.czk.pub/assets/1653911877542.png)

### [](#IO多路复用 "IO多路复用")IO 多路复用

#### [](#场景引入 "场景引入")场景引入

为了更好的理解 IO ，现在假设这样一种场景：一家餐厅

* A 情况：这家餐厅中现在只有一位服务员，并且采用客户排队点餐的方式，就像这样：

  [![](http://blog.czk.pub/assets/1660532354543.png)](http://blog.czk.pub/assets/1660532354543.png)

  每排到一位客户要吃到饭，都要经过两个步骤：

  1.  思考要吃什么
  2.  顾客开始点餐，厨师开始炒菜

  由于餐厅只有一位服务员，因此一次只能服务一位客户，并且还需要等待当前客户思考出结果，这浪费了后续排队的人非常多的时间，效率极低。这就是阻塞 IO。

  当然，为了缓解这种情况，老板完全可以多雇几个人，但这也会增加成本，而在极大客流量的情况下，仍然不会有很高的效率提升

  +++

* B 情况： 这家餐厅中现在只有一位服务员，并且采用客户排队点餐的方式。

  每排到一位客户要吃到饭，都要经过两个步骤：

  1.  思考要吃什么
  2.  顾客开始点餐，厨师开始炒菜

  与 A 情况不同的是，此时服务员会不断询问顾客：“你想吃番茄鸡蛋盖浇饭吗？那滑蛋牛肉呢？那肉末茄子呢？……”

  虽然服务员在不停的问，但是在网络中，这并不会增加数据的就绪速度，主要还是等顾客自己确定。所以，这并不会提高餐厅的效率，说不定还会招来更多差评。这就是非阻塞 IO。

  +++

* C 情况： 这家餐厅中现在只有一位服务员，但是不再采用客户排队的方式，而是顾客自己获取菜单并点餐，点完后通知服务员，就像这样：

  [![](http://blog.czk.pub/assets/1660533483135.png)](http://blog.czk.pub/assets/1660533483135.png)

  每排到一位客户要吃到饭，还是都要经过两个步骤：

  1.  看着菜单，思考要吃什么
  2.  通知服务员，我点好了

  与 A B 不同的是，这种情况服务员不必再等待顾客思考吃什么，只需要在收到顾客通知后，去接收菜单就好。这样相当于餐厅在只有一个服务员的情况下，同时服务了多个人，而不像 A B，同一时刻只能服务一个人。此时餐厅的效率自然就提高了很多。

映射到我们的网络服务中，就是这样：

*   客人：客户端请求

*   点餐内容：客户端发送的实际数据

*   老板：操作系统

*   人力成本：系统资源

*   菜单：文件状态描述符。操作系统对于一个进程能够同时持有的文件状态描述符的个数是有限制的，在 linux 系统中 $ulimit -n 查看这个限制值，当然也是可以 (并且应该) 进行内核参数调整的。

*   服务员：操作系统内核用于 IO 操作的线程 (内核线程)

*   厨师：应用程序线程 (当然厨房就是应用程序进程咯)

*   餐单传递方式：包括了阻塞式和非阻塞式两种。

    *   方法 A: 阻塞 IO

    *   方法 B: 非阻塞 IO

    *   方法 C: 多路复用 IO

### [](#多路复用-IO-的实现 "多路复用 IO 的实现")多路复用 IO 的实现

目前流程的多路复用 IO 实现主要包括四种: `select`、`poll`、`epoll`、`kqueue`。下表是他们的一些重要特性的比较:

<table><thead><tr><th>IO 模型</th><th>相对性能</th><th>关键思路</th><th>操作系统</th><th>JAVA 支持情况</th></tr></thead><tbody><tr><td>select</td><td>较高</td><td>Reactor</td><td>windows/Linux</td><td>支持，Reactor 模式 (反应器设计模式)。Linux 操作系统的 kernels 2.4 内核版本之前，默认使用 select；而目前 windows 下对同步 IO 的支持，都是 select 模型</td></tr><tr><td>poll</td><td>较高</td><td>Reactor</td><td>Linux</td><td>Linux 下的 JAVA NIO 框架，Linux kernels 2.6 内核版本之前使用 poll 进行支持。也是使用的 Reactor 模式</td></tr><tr><td>epoll</td><td>高</td><td>Reactor/Proactor</td><td>Linux</td><td>Linux kernels 2.6 内核版本及以后使用 epoll 进行支持；Linux kernels 2.6 内核版本之前使用 poll 进行支持；另外一定注意，由于 Linux 下没有 Windows 下的 IOCP 技术提供真正的 异步 IO 支持，所以 Linux 下使用 epoll 模拟异步 IO</td></tr><tr><td>kqueue</td><td>高</td><td>Proactor</td><td>Linux</td><td>目前 JAVA 的版本不支持</td></tr></tbody></table>

多路复用 IO 技术最适用的是 “高并发” 场景，所谓高并发是指 1 毫秒内至少同时有上千个连接请求准备好。其他情况下多路复用 IO 技术发挥不出来它的优势。另一方面，使用 JAVA NIO 进行功能实现，相对于传统的 Socket 套接字实现要复杂一些，所以实际应用中，需要根据自己的业务需求进行技术选择。

#### [](#select "select")select

select 是 Linux 最早是由的 I/O 多路复用技术：

linux 中，一切皆文件，socket 也不例外，我们把需要处理的数据封装成 FD，然后在用户态时创建一个 fd_set 的集合（这个集合的大小是要监听的那个 FD 的最大值 + 1，但是大小整体是有限制的 ），这个集合的长度大小是有限制的，同时在这个集合中，标明出来我们要控制哪些数据。

其内部流程：

用户态下：

1.  创建 fd_set 集合，包括要监听的 读事件、写事件、异常事件 的集合
2.  确定要监听的 fd_set 集合
3.  将要监听的集合作为参数传入 select () 函数中，select 中会将 集合复制到内核 buffer 中

内核态：

4.  内核线程在得到 集合后，遍历该集合
5.  没数据就绪，就休眠
6.  当数据来时，线程被唤醒，然后再次遍历集合，标记就绪的 fd 然后将整个集合，复制回用户 buffer 中
7.  用户线程遍历 集合，找到就绪的 fd ，再发起读请求。

[![](http://blog.czk.pub/assets/1653900022580.png)](http://blog.czk.pub/assets/1653900022580.png)

不足之处：

*   集合大小固定为 1024 ，也就是说最多维持 1024 个 socket，在海量数据下，不够用
*   集合需要在 用户 buffer 和内核 buffer 中反复复制，涉及到 用户态和内核态的切换，非常影响性能

#### [](#poll "poll")poll

> poll 模式对 select 模式做了简单改进，但性能提升不明显。

IO 流程：

*   创建 pollfd 数组，向其中添加关注的 fd 信息，数组大小自定义
*   调用 poll 函数，将 pollfd 数组拷贝到内核空间，转链表存储，无上限
*   内核遍历 fd ，判断是否就绪
*   数据就绪或超时后，拷贝 pollfd 数组到用户空间，返回就绪 fd 数量 n
*   用户进程判断 n 是否大于 0, 大于 0 则遍历 pollfd 数组，找到就绪的 fd

**与 select 对比：**

*   select 模式中的 fd_set 大小固定为 1024，而 pollfd 在内核中采用链表，理论上无上限，但实际上不能这么做，因为的监听 FD 越多，每次遍历消耗时间也越久，性能反而会下降

[![](http://blog.czk.pub/assets/1653900721427.png)](http://blog.czk.pub/assets/1653900721427.png)

#### [](#epoll "epoll")epoll

> epoll 模式是对 select 和 poll 的改进，它提供了三个函数：eventpoll 、epoll_ctl 、epoll_wait

*   eventpoll 函数内部包含了两个东西 :

    *   红黑树 ：用来记录所有的 fd

    *   链表 ： 记录已就绪的 fd

*   epoll_ctl 函数 ，将要监听的 fd 添加到 红黑树 上去，并且给每个 fd 绑定一个监听函数，当 fd 就绪时就会被触发，这个监听函数的操作就是 将这个 fd 添加到 链表中去。

*   epoll_wait 函数，就绪等待。一开始，用户态 buffer 中创建一个空的 events 数组，当就绪之后，我们的回调函数会把 fd 添加到链表中去，当函数被调用的时候，会去检查链表（当然这个过程需要参考配置的等待时间，可以等一定时间，也可以一直等），如果链表中没有有 fd 则 fd 会从红黑树被添加到链表中，此时再将链表中的的 fd 复制到 用户态的空 events 中，并且返回对应的操作数量，用户态此时收到响应后，会从 events 中拿到已经准备好的数据，在调用 读方法 去拿数据。

#### [](#总结 "总结")总结

select 模式存在的三个问题：

*   能监听的 FD 最大不超过 1024
*   每次 select 都需要把所有要监听的 FD 都拷贝到内核空间
*   每次都要遍历所有 FD 来判断就绪状态

poll 模式的问题：

*   poll 利用链表解决了 select 中监听 FD 上限的问题，但依然要遍历所有 FD，如果监听较多，性能会下降

epoll 模式中如何解决这些问题的？

*   基于 epoll 实例中的红黑树保存要监听的 FD，理论上无上限，而且增删改查效率都非常高
*   每个 FD 只需要执行一次 epoll_ctl 添加到红黑树，以后每次 epol_wait 无需传递任何参数，无需重复拷贝 FD 到内核空间
*   利用 ep_poll_callback 机制来监听 FD 状态，无需遍历所有 FD，因此性能不会随监听的 FD 数量增多而下降

### [](#基于-epoll-的服务器端流程 "基于 epoll 的服务器端流程")基于 epoll 的服务器端流程

一张图搞定：

[![](http://blog.czk.pub/assets/1653902845082.png)](http://blog.czk.pub/assets/1653902845082.png)

我们来梳理一下这张图

1.  服务器启动以后，服务端会去调用 epoll_create，创建一个 epoll 实例，epoll 实例中包含两个数据

    1.  红黑树（为空）：rb_root 用来去记录需要被监听的 FD
    2.  链表（为空）：list_head，用来存放已经就绪的 FD
2.  创建好了之后，会去调用 epoll_ctl 函数，此函数会会将需要监听的 fd 添加到 rb_root 中去，并且对当前这些存在于红黑树的节点设置回调函数。

3.  当这些被监听的 fd 一旦准备就绪，与之相关联的回调函数就会被调用，而调用的结果就是将红黑树的 fd 添加到 list_head 中去 (但是此时并没有完成)

4.  fd 添加完成后，就会调用 epoll_wait 函数，这个函数会去校验是否有 fd 准备就绪（因为 fd 一旦准备就绪，就会被回调函数添加到 list_head 中），在等待了一段时间 (可以进行配置)。

5.  如果等够了超时时间，则返回没有数据，如果有，则进一步判断当前是什么事件，如果是建立连接事件，则调用 accept () 接受客户端 socket ，拿到建立连接的 socket ，然后建立起来连接，如果是其他事件，则把数据进行写出。

### [](#五种网络模型对比 "五种网络模型对比")五种网络模型对比

最后用一幅图，来说明他们之间的区别

[![](http://blog.czk.pub/assets/1653912219712.png)](http://blog.czk.pub/assets/1653912219712.png)

[](#redis-通信协议 "redis 通信协议")redis 通信协议
--------------------------------------

### [](#RESP协议 "RESP协议")RESP 协议

Redis 是一个 CS 架构的软件，通信一般分两步（不包括 pipeline 和 PubSub）：

*   客户端（client）向服务端（server）发送一条命令，服务端解析并执行命令，
*   返回响应结果给客户端，因此客户端发送命令的格式、服务端响应结果的格式必须有一个规范，这个规范就是通信协议。

而在 Redis 中采用的是 RESP（Redis Serialization Protocol）协议：

*   Redis 1.2 版本引入了 RESP 协议
*   Redis 2.0 版本中成为与 Redis 服务端通信的标准，称为 RESP2
*   Redis 6.0 版本中，从 RESP2 升级到了 RESP3 协议，增加了更多数据类型并且支持 6.0 的新特性–客户端缓存

但目前，默认使用的依然是 RESP2 协议。在 RESP 中，通过首字节的字符来区分不同数据类型，常用的数据类型包括 5 种：

*   单行字符串：首字节是 ‘+’ ，后面跟上单行字符串，以 CRLF（ “\r\n” ）结尾。例如返回”OK”： “+OK\r\n”

*   错误（Errors）：首字节是 ‘-’ ，与单行字符串格式一样，只是字符串是异常信息，例如：”-Error message\r\n”

*   数值：首字节是 ‘:’ ，后面跟上数字格式的字符串，以 CRLF 结尾。例如：”:10\r\n”

*   多行字符串：首字节是 ‘$’ ，表示二进制安全的字符串，最大支持 512MB：

    *   如果大小为 0，则代表空字符串：”$0\r\n\r\n”
    *   如果大小为 - 1，则代表不存在：”$-1\r\n”
*   数组：首字节是 ‘*’，后面跟上数组元素个数，再跟上元素，元素数据类型不限 :

[![](http://blog.czk.pub/assets/1653982993020.png)](http://blog.czk.pub/assets/1653982993020.png)

### [](#自定义Redis的客户端 "自定义Redis的客户端")自定义 Redis 的客户端

Redis 支持 TCP 通信，因此我们可以使用 Socket 来模拟客户端，与 Redis 服务端建立连接：

```
public class Main {

    static Socket s;
    static PrintWriter writer;
    static BufferedReader reader;

    public static void main(String[] args) {
        try {
            
            String host = "192.168.150.101";
            int port = 6379;
            s = new Socket(host, port);
            
            writer = new PrintWriter(new OutputStreamWriter(s.getOutputStream(), StandardCharsets.UTF_8));
            reader = new BufferedReader(new InputStreamReader(s.getInputStream(), StandardCharsets.UTF_8));

            
            
            sendRequest("auth", "123321");
            Object obj = handleResponse();
            System.out.println("obj = " + obj);

            
            sendRequest("set", "name", "虎哥");
            
            obj = handleResponse();
            System.out.println("obj = " + obj);

            
            sendRequest("get", "name");
            
            obj = handleResponse();
            System.out.println("obj = " + obj);

            
            sendRequest("mget", "name", "num", "msg");
            
            obj = handleResponse();
            System.out.println("obj = " + obj);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            
            try {
                if (reader != null) reader.close();
                if (writer != null) writer.close();
                if (s != null) s.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private static Object handleResponse() throws IOException {
        
        int prefix = reader.read();
        
        switch (prefix) {
            case '+': 
                return reader.readLine();
            case '-': 
                throw new RuntimeException(reader.readLine());
            case ':': 
                return Long.parseLong(reader.readLine());
            case '$': 
                
                int len = Integer.parseInt(reader.readLine());
                if (len == -1) {
                    return null;
                }
                if (len == 0) {
                    return "";
                }
                
                return reader.readLine();
            case '*':
                return readBulkString();
            default:
                throw new RuntimeException("错误的数据格式！");
        }
    }

    private static Object readBulkString() throws IOException {
        
        int len = Integer.parseInt(reader.readLine());
        if (len <= 0) {
            return null;
        }
        
        List<Object> list = new ArrayList<>(len);
        
        for (int i = 0; i < len; i++) {
            list.add(handleResponse());
        }
        return list;
    }

    
    private static void sendRequest(String ... args) {
        writer.println("*" + args.length);
        for (String arg : args) {
            writer.println("$" + arg.getBytes(StandardCharsets.UTF_8).length);
            writer.println(arg);
        }
        writer.flush();
    }
}
```

[](#redis-内存回收 "redis 内存回收")redis 内存回收
--------------------------------------

### [](#过期key处理 "过期key处理")过期 key 处理

Redis 之所以性能强，最主要的原因就是基于内存存储。然而单节点的 Redis 其内存大小不宜过大，会影响持久化或主从同步性能。  
我们可以通过修改配置文件来设置 Redis 的最大内存：

```
maxmemory 1gb
```

当内存使用达到上限时，就无法存储更多数据了。为了解决这个问题，Redis 提供了一些策略实现内存回收：

先要了解的是：redis 是一个存储键值数据库系统，那它源码中是如何存储所有键值对的呢？

> Redis 本身是一个典型的 key-value 内存存储数据库，因此所有的 key、value 都保存在之前学习过的 Dict 结构中。不过在其 database 结构体中，有两个 Dict：一个用来记录 key-value；另一个用来记录 key-TTL。

[![](http://blog.czk.pub/assets/1653983423128.png)](http://blog.czk.pub/assets/1653983423128.png)

内部结构

[![](http://blog.czk.pub/assets/1653983606531.png)](http://blog.czk.pub/assets/1653983606531.png)

*   dict 是 hash 结构，用来存放所有的 键值对
*   expires 也是 hash 结构，用来存放所有设置了 过期时间的 键值对，不过它的 value 值是过期时间

这里有两个问题需要我们思考：

*   Redis 是如何知道一个 key 是否过期呢？
*   利用两个 Dict 分别记录 key-value 对及 key-ttl 对，是不是 TTL 到期就立即删除了呢？

#### [](#惰性删除 "惰性删除")惰性删除

> 惰性删除：顾明思议并不是在 TTL 到期后就立刻删除，而是在访问一个 key 的时候，检查该 key 的存活时间，如果已经过期才执行删除。

#### [](#周期删除 "周期删除")周期删除

周期删除：顾明思议是通过一个定时任务，周期性的抽样部分过期的 key，然后执行删除。执行周期有两种：

*   Redis 服务初始化函数 initServer () 中设置定时任务，按照 server.hz 的频率来执行过期 key 清理，模式为 SLOW
*   Redis 的每个事件循环前会调用 beforeSleep () 函数，执行过期 key 清理，模式为 FAST

SLOW 模式规则：

*   执行频率受 server.hz 影响，默认为 10，即每秒执行 10 次，每个执行周期 100ms。
*   执行清理耗时不超过一次执行周期的 25%. 默认 slow 模式耗时不超过 25ms
*   逐个遍历 db，逐个遍历 db 中的 bucket，抽取 20 个 key 判断是否过期
*   如果没达到时间上限（25ms）并且过期 key 比例大于 10%，再进行一次抽样，否则结束

FAST 模式规则（过期 key 比例小于 10% 不执行 ）：

*   执行频率受 beforeSleep () 调用频率影响，但两次 FAST 模式间隔不低于 2ms
*   执行清理耗时不超过 1ms
*   逐个遍历 db，逐个遍历 db 中的 bucket，抽取 20 个 key 判断是否过期
*   如果没达到时间上限（1ms）并且过期 key 比例大于 10%，再进行一次抽样，否则结束

### [](#内存淘汰策略 "内存淘汰策略")内存淘汰策略

> 内存淘汰：就是当 Redis 内存使用达到设置的上限时，主动挑选部分 key 删除以释放更多内存的流程。

淘汰策略

Redis 支持 8 种不同策略来选择要删除的 key：

*   noeviction： 不淘汰任何 key，但是内存满时不允许写入新数据，默认就是这种策略。
*   volatile-ttl： 对设置了 TTL 的 key，比较 key 的剩余 TTL 值，TTL 越小越先被淘汰
*   allkeys-random：对全体 key ，随机进行淘汰。也就是直接从 db->dict 中随机挑选
*   volatile-random：对设置了 TTL 的 key ，随机进行淘汰。也就是从 db->expires 中随机挑选。
*   allkeys-lru： 对全体 key，基于 LRU 算法进行淘汰
*   volatile-lru： 对设置了 TTL 的 key，基于 LRU 算法进行淘汰
*   allkeys-lfu： 对全体 key，基于 LFU 算法进行淘汰
*   volatile-lfu： 对设置了 TTL 的 key，基于 LFI 算法进行淘汰  
    比较容易混淆的有两个：
    *   LRU（Least Recently Used），最少最近使用。用当前时间减去最后一次访问时间，这个值越大则淘汰优先级越高。
    *   LFU（Least Frequently Used），最少频率使用。会统计每个 key 的访问频率，值越小淘汰优先级越高。

Redis 的数据都会被封装为 RedisObject 结构：

[![](http://blog.czk.pub/assets/1653984029506.png)](http://blog.czk.pub/assets/1653984029506.png)

LFU 的访问次数之所以叫做逻辑访问次数，是因为并不是每次 key 被访问都计数，而是通过运算：

*   生成 0~1 之间的随机数 R
*   计算 (旧次数 * lfu_log_factor + 1)，记录为 P
*   如果 R < P ，则计数器 + 1，且最大不超过 255
*   访问次数会随时间衰减，距离上一次访问时间每隔 lfu_decay_time 分钟，计数器 -1