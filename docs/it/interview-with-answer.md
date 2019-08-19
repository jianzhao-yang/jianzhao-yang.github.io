## Java

### - Java 基础

1. JDK 和 JRE 有什么区别？

2. == 和 equals 的区别是什么？

3.  两个对象的 hashCode()相同，则 equals()也一定为 true，对吗？

4.  final 在 java 中有什么作用？

5. java 中的 Math.round(-1. 5)  等于多少？

6. String 属于基础的数据类型吗？

7. java 中操作字符串都有哪些类？它们之间有什么区别？

8. String str="i"与 String str=new String(“i”)一样吗？

9. 如何将字符串反转？

10. String 类的常用方法都有那些？

11. 抽象类必须要有抽象方法吗？

12. 普通类和抽象类有哪些区别？

13. 抽象类能使用 final 修饰吗？

14. 接口和抽象类有什么区别？

15. java 中 IO 流分为几种？

16. BIO、NIO、AIO 有什么区别？

17. Files的常用方法都有哪些？

### - 容器

18. java 容器都有哪些？

19. Collection 和 Collections 有什么区别？

20. List、Set、Map 之间的区别是什么？

21. HashMap 和 Hashtable 有什么区别？

22. 如何决定使用 HashMap 还是 TreeMap？

23. 说一下 HashMap 的实现原理？

24. 说一下 HashSet 的实现原理？

25. ArrayList 和 LinkedList 的区别是什么？

26. 如何实现数组和 List 之间的转换？

27. ArrayList 和 Vector 的区别是什么？

28. Array 和 ArrayList 有何区别？

29. 在 Queue 中 poll()和 remove()有什么区别？

30. 哪些集合类是线程安全的？

31. 迭代器 Iterator 是什么？

32. Iterator 怎么使用？有什么特点？

33. Iterator 和 ListIterator 有什么区别？

34. 怎么确保一个集合不能被修改？

### - 多线程

35. [什么是线程安全?](https://www.jianshu.com/p/fc61770094a4)

36. 并行和并发有什么区别？

    · 并发是一段时间内同时发生，并行是某个时间点同时运行

37. 线程和进程的区别？

    - 进程是资源分配的最小单位，线程是程序执行的最小单位。
    - 进程有自己的独立地址空间，每启动一个进程，系统就会为它分配地址空间，建立数据表来维护代码段、堆栈段和数据段，这种操作非常昂贵。而线程是共享进程中的数据的，使用相同的地址空间，因此CPU切换一个线程的花费远比进程要小很多，同时创建一个线程的开销也比进程要小很多。
    - 线程之间的通信更方便，同一进程下的线程共享全局变量、静态变量等数据，而进程之间的通信需要以通信的方式（IPC)进行。不过如何处理好同步与互斥是编写多线程程序的难点。

38. 守护线程是什么？

    · 用于服务其他用户线程，当所有用户线程退出时，守护线程也自动退出

39. 创建线程有哪几种方式？

    · 继承Thread类

    · 实现Runnable接口

    · 实现Callable接口

40. 说一下 runnable 和 callable 有什么区别？

    · 实现Runnable接口的线程执行后无返回值

    · 实现Callable接口的线程执行后有返回值

41. 线程有哪些状态？

    ![](images\threadstates.jpg)

    · new、runnable、running、blocked、waiting/timed-waiting、dead

42. [sleep()与wait()区别](https://blog.csdn.net/linfanhehe/article/details/78737685)？

    sleep会自动唤醒，wait不会；

    sleep不会释放锁，wait会；

    wait只能在同步方法块中使用，sleep可以在任何地方使用；

    sleep属于Thread类，wait属于Object类；

43. notify()和 notifyAll()有什么区别？

    · notify只会通知一个在等待的对象，而notifyAll会通知所有在等待的对象

    · notifyAll唤醒的线程进入锁池后继续竞争，直到有一个获取锁之后其它线程继续等待

44. 线程的 run()和 start()有什么区别？

    · run只是普通方法，不会单起线程

45. 创建线程池有哪几种方式？

    · 使用自带的Executors实现

    ```java
    public class Executors {
    	public static ExecutorService newFixedThreadPool(int nThreads)
    	public static ExecutorService newCachedThreadPool()
    	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
    	public static ExecutorService newSingleThreadExecutor()
    }
    ```

    · 手动创建

    ```java
    ThreadPoolExecutor(int corePoolSize, // 核心池大小
                                  int maximumPoolSize,// 最大池大小
                                  long keepAliveTime,// 存活时间
                                  TimeUnit unit,// 时间单位
                                  BlockingQueue<Runnable> workQueue,//缓存队列
                                  ThreadFactory threadFactory,//线程工厂
                                  RejectedExecutionHandler handler)//队列满之后处理逻辑
    ```

46. [线程池都有哪些状态？](https://blog.csdn.net/u011389515/article/details/80656813)

    · Running、ShutDown、Stop、Tidying、Terminated

47. 线程池中 submit()和 execute()方法有什么区别？

    · submit()方法，可以提供Future < T > 类型的返回值；executor()方法，无返回值

    · excute方法会抛出异常；sumbit方法不会抛出异常。除非调用Future.get()

    · excute入参Runnable；submit入参可以为Callable，也可以为Runnable

48. 在 java 程序中怎么保证多线程的运行安全？

    ```java
    线程的安全性问题体现在：
    原子性：一个或者多个操作在 CPU 执行的过程中不被中断的特性
    可见性：一个线程对共享变量的修改，另外一个线程能够立刻看到
    有序性：程序执行的顺序按照代码的先后顺序执行
     
    导致原因：
    缓存导致的可见性问题
    线程切换带来的原子性问题
    编译优化带来的有序性问题
    
    解决办法：
    JDK Atomic开头的原子类、synchronized、LOCK，可以解决原子性问题
    synchronized、volatile、LOCK，可以解决可见性问题
    Happens-Before 规则可以解决有序性问题
    ```

49. 多线程锁的升级原理是什么？

    · 从轻到重： [无锁 -> 偏向锁(独占) -> 轻量级锁(自旋等待) -> 重量级锁(MutexLock互斥锁)](https://www.liangzl.com/get-article-detail-135712.html)

50. 什么是死锁？怎么防止死锁？

    · [死锁](https://www.cnblogs.com/bopo/p/9228834.html)

51. ThreadLocal 是什么？有哪些使用场景？

    · ThreadLocal是一种解决并发问题的思路，即为每个线程都创建一个新的副本，从而保证线程间不会互相干扰

    · 实现原理为：Thread类里保存一个ThreadLocalMap，该线程的所有ThreadLocal对象都保存在这里，随线程退出而清除

52. [说一下synchronized底层实现原理](https://baijiahao.baidu.com/s?id=1612142459503895416&wfr=spider&for=pc)

53. synchronized 和 volatile 的区别是什么？

    · volatile只能作用于变量，使用范围较小。synchronized可以用在变量、方法、类、同步代码块等，使用范围比较广。 
    · volatile只能保证可见性和有序性，不能保证原子性。而可见性、有序性、原子性synchronized都可以包证。 
    · volatile不会造成线程阻塞。synchronized可能会造成线程阻塞。

54. synchronized 和 Lock/ReentrantLock  有什么区别？

    · 实现层面不一样。synchronized 是 Java 关键字，JVM层面 实现加锁和释放锁；Lock 是一个接口，在代码层面实现加锁和释放锁
    · 是否自动释放锁。synchronized 在线程代码执行完或出现异常时自动释放锁；Lock 不会自动释放锁，需要再 finally {} 代码块显式地中释放锁
    · 是否一直等待。synchronized 会导致线程拿不到锁一直等待；Lock 可以设置尝试获取锁或者获取锁失败一定时间超时
    · 获取锁成功是否可知。synchronized 无法得知是否获取锁成功；Lock 可以通过 tryLock 获得加锁是否成功
    · 功能复杂性。synchronized 加锁可重入、不可中断、非公平；Lock 可重入、可判断、可公平和不公平、细分读写锁提高效率

    · [ReentrantLock使用Condition进行多条件应用

55. [说一下 atomic 的原理？](https://www.jianshu.com/p/84c75074fa03)

    · 通过UnSafe类的CAS操作保证原子性

### - 反射

57. 什么是反射？

    · Java 反射机制是在运行状态中，对于任意一个类，都能够获得这个类的所有属性和方法，对于任意一个对象都能够调用它的任意一个属性和方法。这种在运行时动态的获取信息以及动态调用对象的方法的功能称为Java 的反射机制

58. 哪里用到反射机制？

    1. JDBC中，利用反射动态加载了数据库驱动程序。
    2. Web服务器中利用反射调用了Sevlet的服务方法。
    3. Eclispe等开发工具利用反射动态刨析对象的类型与结构，动态提示对象的属性和方法。
    4. 很多框架都用到反射机制，注入属性，调用方法，如Spring。

59. Java反射机制的作用

    1. 在运行时判断任意一个对象所属的类
    2. 在运行时构造任意一个类的对象
    3. 在运行时判断任意一个类所具有的成员变量和方法
    4. 在运行时调用任意一个对象的方法
    5. 在运行时创建新类
    6. 在运行时修改对象的属性值

60. 反射获取字节码的方式？

    · obj.getClass() 方法

    · 类名.class 静态属性

    · Class.forName(类全名)

61. 什么是 java 序列化？什么情况下需要序列化？

    1. 对象序列化，将对象中的数据编码为字节序列的过程。

    2. 以下情况：

       · 想把的内存中的对象状态保存到缓存或磁盘中时；

       · 想用套接字在网络上传送对象的时候；

       · 想通过RMI（远程方法调用）传输对象的时候；

62. 动态代理是什么？有哪些应用？

    · 动态代理是一种在运行时动态地创建代理对象，动态地处理代理方法调用的机制。

    · 用于AOP、RPC；增加日志、权限、事务等切面功能

63. 怎么实现动态代理？

    · JDK动态代理： 反射机制；实现InvocationHandler 接口，再用Proxy.newProxyInstance()生成代理接口实现类

    · Cglib：ASM字节码操纵机制；实现MethodInterceptor 接口，使用Enhancer 创建代理类

    · javassist：静态生成字节码性能更好，代理时性能下降

### - 对象拷贝

61. 为什么要使用克隆？

    · 对象复制，对对象的修改不影响原对象

62. 如何实现对象克隆？

    · 实现Cloneable接口，重写clone()方法

63. 深拷贝和浅拷贝区别是什么？

    · 浅拷贝---能复制变量，如果对象内还有对象，则只能复制对象的地址

    · 深拷贝---能复制变量，也能复制当前对象的 内部对象

### - 异常

74. throw 和 throws 的区别？

    · throws 作用在方法签名上，用于指出方法可能抛出的异常

    · throw作用与代码块中，用于抛出指定类型的异常

75. final、finally、finalize 有什么区别？

    · final可以作用于类、方法、变量；作用于类和方法表明其不可继承/重写；作用于变量表示变量值或引用不能改变

    · finally是try-catch-finally语句中的一部分，用于在任何情况下都执行一段代码，如释放资源等

    · finalize是Object 类的方法，用于在回收对象时执行相关操作；一个对象只能使用finalize复活一次

76. try-catch-finally 中哪个部分可以省略？

    · finally

77. try-catch-finally 中，如果 catch 中 return 了，finally 还会执行吗？

    · 会，finally会先于其他语句的return执行

78. 常见的异常类有哪些？

    · NullPointerException、ClassCastException、ArithmeticExecption、ArrayIndexOutOfBoundsException、FileNotFoundException、IOException、UnsupportedOperationException

### -  注解

[Java自定义注解](https://www.cnblogs.com/huojg-21442/p/7239846.html)
	
[springboot自定义注解实现aop](https://www.cnblogs.com/hhhshct/p/8428045.html)


### - JVM

194. 说一下 jvm 的主要组成部分？及其作用？

     ![jvm](images\jvm.png)

195. 说一下 jvm 运行时数据区？

     · [JVM运行时数据区和Java内存模型](https://www.jianshu.com/p/2cdd069a4d5f)

196. 说一下堆栈的区别？

     · 堆是所有线程共享一个的，栈是每个线程都有一个的

     · 堆用来存放对象和数组，栈用来存放方法和局部变量

197. 队列和栈是什么？有什么区别？

     · 队列先进先出，栈先进后出

     · 队列遍历速度更快

198. 什么是双亲委派模型？

     ![类加载器结构](/images/classloader.png)

     · 先是自下而上的请求父类加载，再自上而下的尝试加载

     [为什么说java spi破坏双亲委派模型？](https://www.zhihu.com/question/49667892?sort=created)

199. 说一下类加载的执行过程？

     ![类加载过程](/images/load.png)

     [深入浅出Java类加载过程](https://www.cnblogs.com/luohanguo/p/9469851.html)

200. 怎么判断对象是否可以被回收？

     · 根据GCRoots进行可达性分析，不可达即可回收

     · GCRoots:栈中引用的对象、静态属性引用的对象、常量池中引用的对象

201. java 中都有哪些引用类型？

     · 强引用：正常引用

     · 软引用： OOM之前会清除

     · 弱引用： GC即回收

     · 虚引用： 待清除的对象，不能再获取，需要与引用队列一起使用

     · [四种引用类型](https://www.cnblogs.com/liyutian/p/9690974.html)

202. 说一下 jvm 有哪些垃圾回收算法？

     · 分代回收算法

     · 复制算法

     · 标记清除算法

     · 标记整理算法

 [JVM的垃圾回收机制 总结(垃圾收集、回收算法、垃圾回收器)](https://www.cnblogs.com/aspirant/p/8662690.html)

203. 说一下 jvm 有哪些垃圾回收器？

     · 新生代的垃圾收集器有：Serial收集器、ParNew收集器、Parallel Scavenge收集器

     · 老年代的垃圾收集器有：Serial Old收集器、Parallel Old收集器、CMS收集器、G1收集器

204. 详细介绍一下 CMS 垃圾回收器？https://blog.csdn.net/mc90716/article/details/80158138

205. 新生代垃圾回收器和老生代垃圾回收器都有哪些？有什么区别？

206. 简述分代垃圾回收器是怎么工作的？

     · 对象出生在新生代Eden，Eden不足时进行minorGC，将存活对象转移到S1，同时年龄+1；

     · S1满后GC，存活对象转移到S2，S1和S2往复轮转，直到年龄到达晋升标准

     · 晋升进入老年代，新生代放不下的大对象直接晋升到老年代；

     · 新生代GC内存不足触发对象晋升，对象晋升老年代内存不足触发老年代GC

207. [JVM中的STW和CMS](https://www.cnblogs.com/williamjie/p/9222839.html)

208. 说一下 jvm 调优的工具？

     · Windows：JVisualVM 工具集

     · jps 查看运行的Java程序和jvm内部pid

     · jstack 主要用来查看某个Java进程内的线程堆栈信息

     · jmap 用来查看堆内存使用状况

     · jstat: 看看各个区内存和GC的情况

209. 常用的 jvm 调优的参数都有哪些？

     https://cloud.tencent.com/developer/article/1198524

210. [JVM调优总结](https://www.cnblogs.com/andy-zhou/p/5327288.html)

### - [Java8新特性]([https://snailclimb.gitee.io/javaguide/#/java/What's%20New%20in%20JDK8/Java8Tutorial](https://snailclimb.gitee.io/javaguide/#/java/What's New in JDK8/Java8Tutorial))

## Web

### - Java Web

64. jsp 和 servlet 有什么区别？

65. jsp 有哪些内置对象？作用分别是什么？

66. 说一下 jsp 的 4  种作用域？

67. session 和 cookie 有什么区别？

    · session保存在服务端，cookie保存在客户端

68. 说一下 session 的工作原理？

    · 客户端每次请求会在cookie中带上sessionId，服务端接受请求后先解析sessionId，再根据id到缓存中查找session信息，以供本次会话使用

69. 如果客户端禁止 cookie 能实现 session 还能用吗？

    · 不能，没有sessionId就无法找到对应session

    · 可以**通过其他方法**在禁用 cookie 的情况下，可以继续使用session：

    1. 使用url保存sessionId

    	2. 使用header保存sessionId
     	3. 每次返回时以参数的形式返回sessionId

70. spring mvc 和 struts 的区别是什么？

71. 如何避免 sql 注入？

    · 使用PreparedStatement

    · 过滤和转义特殊字符

72. 什么是 XSS 攻击，如何避免？

    · XSS（Cross Site Scripting），即跨站脚本攻击，通过在用户端注入恶意的可运行脚本，若服务器端对用户输入不进行处理，直接将用户输入输出到浏览器，然后浏览器将会执行用户注入的脚本。

    · 即输入一段脚本保存到server，其他用户在看到这段脚本的网页时会自动执行

    · 对用户输入的特殊字符进行转义

73. 什么是 CSRF 攻击，如何避免？

    ·跨站请求伪造，在另一个网站的网页下伪造表单提交到被攻击的网站（利用了浏览器cookie共享的机制获取登录态） 

### - 网络

79. http 响应码 301  和 302  代表的是什么？有什么区别？

    · 30x都是重定向： 301永久重定向（建议以后直接访问新的url），302临时重定向（建议沿用原url）；

    [状态码详解](http://www.webmasterhome.cn/httpstatus/)

80. forward 和 redirect 的区别？

    · forward是服务器端内部转发，不可跨站；redirect是客户端重定向，可跨站

    · forward浏览器地址不变，redirect会改变

    · forward会共享request相关 信息，redirect不会

    · forward是服务端行为，redirect是客户端行为

81. 简述 tcp 和 udp的区别？

    · TCP基于连接，UDP基于无连接。

    · TCP对系统资源要求高，UDP少。

    · TCP是基于字节流的，UDP是数据报文模式。

    · TCP复杂，UDP简单。

82. tcp 为什么要三次握手，两次不行吗？为什么？

    · A -> B: Hello B，你在吗？我想和你建立连接（SYN）

    · B -> A: Hi A，我在(ACK)，我也想和你建立连接(SYN)  (此时A知道自己能发送也能接收了，B只知道自己能接收)

    · A -> B: Hi B,我也在(ACK) (此时B才能确定自己能发送信息) 

83. Tcp的四次挥手

    · A -> B : Hello B,我想和你断开连接(FIN)

    · B -> A : Hi A,我知道了(ACK),但我可能还有些东西要发给你，请不要立马断开

    · A -> B : Hello A,我已经处理完了(ACK)，可以断开了(FIN)

    · B -> A : Hi B,我知道了(ACK)，我马上就和你断开

    [TCP的三次握手与四次挥手](https://blog.csdn.net/qq_38950316/article/details/81087809)

84. 说一下 tcp 粘包是怎么产生的？

    · tcp是基于连接的，发送的消息直接没有明确的界限；udp是基于消息的；tcp有ack确认接收成功(成功发送方才删除)，udp无

    · 因为在cpu和网卡之间存在缓冲区；发送端一次发送的数据大于缓冲区大小会拆包，小于则会粘包；接收端亦是如此

    

85. 粘包/拆包的解决方案？

    · 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容。

    · 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息。

    · 设置消息边界，服务端从网络流中按消息边界分离出消息内容。

86. OSI 的七层模型都有哪些？

    ![四层网络模型](https://upload-images.jianshu.io/upload_images/2179030-3694fe2b18ebe05f.png)

87. get 和 post 请求有哪些区别？

    · GET产生一个TCP数据包；POST产生两个TCP数据包。

    · get请求会被浏览器主动缓存，post不会

    · get请求长度有限制，具体看浏览器和服务器限制

    · get必须使用url编码

88. 如何实现跨域？

    · [注解@CrossOrigin解决跨域问题](https://www.cnblogs.com/mmzs/p/9167743.html)

89. 说一下 JSONP 实现原理？

    ·  [彻底弄懂jsonp原理及实现方法](https://www.cnblogs.com/soyxiaobi/p/9616011.html)

### - netty




## 框架

### - Spring/Spring MVC

90. 为什么要使用 spring？

    - 解耦 IOC
    - 面向切面编程 AOP
    - 方便集成其他框架
    - 声明式事务

91. 解释一下什么是 aop？

    - AOP即面向切面编程，是OOP编程的有效补充。
    - AOP技术恰恰相反，它利用一种称为"横切"的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，并将其命名为"Aspect"，即切面。所谓"切面"，简单说就是那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块之间的耦合度，并有利于未来的可操作性和可维护性

92. AOP的相关概念

    [AOP相关概念](https://www.cnblogs.com/songanwei/p/9417343.html)

93. 解释一下什么是 ioc？

    - IOC是Inversion of Control的缩写，翻译成“控制反转”。
    - 使用一个第三方来管理对象之间的依赖关系,从而解决各个对象之间紧密依赖的情况;达到解耦的作用
    - 哪些方面的控制被反转了呢? 获得依赖对象的过程被反转了. 
    - 所谓依赖注入，就是由IOC容器在运行期间，动态地将某种依赖关系注入到对象之中

94. spring 有哪些主要模块？

    主要七大模块介绍

    1. Spring AOP  面相切面编程

    2. Spring ORM  Hibernate|mybatis|JDO

    3. Spring Core  提供bean工厂 IOC

    4. Spring Dao  JDBC支持

    5. Spring Context  提供了关于UI支持,邮件支持等

    6. Spring Web 提供了web的一些工具类的支持

    7. Spring MVC  提供了web mvc , webviews , jsp ,pdf ,export

95. spring 常用的注入方式有哪些？

    - 构造方法注入
    - setter注入
    - 注解注入(@AutoWried按类型注入,@Resource按名称(先)和类型(后)注入)

96. spring 中的 bean 是线程安全的吗？

    - 不是

97. spring 支持几种 bean 的作用域？

    - singleton：单例模式
    - prototype：原型模式，每次通过容器的getBean方法获取prototype定义的Bean时，都将产生一个新的Bean实例
    - request：对于每次HTTP请求，使用request定义的Bean都将产生一个新实例，即每次HTTP请求将会产生不同的Bean实例。只有在Web应用中使用Spring时，该作用域才有效
    - session：对于每次HTTP Session，使用session定义的Bean都将产生一个新实例。同样只有在Web应用中使用Spring时，该作用域才有效
    - globalsession：每个全局的HTTP Session，使用session定义的Bean都将产生一个新实例。典型情况下，仅在使用portlet context的时候有效。同样只有在Web应用中使用Spring时，该作用域才有效

98. spring 自动装配 bean 有哪些方式？

    - xml和注解

99. spring 事务实现方式有哪些？

    - 基于 TransactionProxyFactoryBean的声明式事务管理

    - 基于 @Transactional 的声明式事务管理
    - 基于Aspectj AOP配置事务

100. 说一下 spring 的事务传播和隔离机制？

     [Spring 事务传播特性和隔离级别](https://www.jianshu.com/p/d42b8c9aa950)

101. 脏读、幻读和不可重复读

     - 脏读： 读到未提交的数据
     - 不可重复读： 两次读取同一条数据不一致（数据被更改、删除了）

     - 幻读： 两次读取数据量不一致 （数据量增加了）

102. 说一下 spring mvc 运行流程？

     [SpringMvc工作流程]([https://snailclimb.gitee.io/javaguide/#/system-design/framework/spring/SpringMVC-Principle?id=springmvc-%e5%b7%a5%e4%bd%9c%e5%8e%9f%e7%90%86%ef%bc%88%e9%87%8d%e8%a6%81%ef%bc%89](https://snailclimb.gitee.io/javaguide/#/system-design/framework/spring/SpringMVC-Principle?id=springmvc-工作原理（重要）))

103. spring mvc 有哪些组件？

     　　1. DispatcherServlet　　请求入口

     　　2. HandlerMapping　　  请求派发,负责请求和控制器建立一一对应的关系

     　　3. Controller　　　　　  处理器

     　　4. ModelAndView　　　  封装模型信息和视图信息

     　　5. ViewResolver　　　　视图处理器,定位页面

104. SpringMVC启动流程

     1. 初始化应用部署描述文件中每一个listener。

     2. 初始化ServletContextListener实现类，调用contextInitialized（）方法。

     3. 初始化应用部署描述文件中每一个filter，并执行每一个的init（）方法。

     4. 按照顺序<load-on-startup>来初始化servlet，并执行init（）方法。

105. @RequestMapping 的作用是什么？

     - 用于映射url到控制器类或一个特定的处理程序方法

106. @Autowired 的作用是什么？

     - 可以对类成员变量、方法及构造函数进行标注，让 spring 完成 bean 自动装配的工作。

### - Spring Boot/Spring Cloud

104. 什么是 spring boot？

105. 为什么要用 spring boot？

106. spring boot 核心配置文件是什么？

107. spring boot 配置文件有哪几种类型？它们有什么区别？

108. spring boot 有哪些方式可以实现热部署？

109. jpa 和 hibernate 有什么区别？

110. 什么是 spring cloud？

111. spring cloud 断路器的作用是什么？

112. spring cloud 的核心组件有哪些？



### - Mybatis

125. mybatis 中 #{}和 ${}的区别是什么？

     - #{}是预编译处理,${}是字符串替换

126. mybatis 有几种分页方式？

     - limit
     - query(String userName, RowBounds rowBounds)
     - 实现拦截器
     - PageHelper

127. RowBounds 是一次性查询全部结果吗？为什么？

     - RowBounds是一次性查询全部结果
     - 从RowBounds源码看出，RowBounds最大数据量为Integer.MAX_VALUE(2147483647)

128. mybatis 逻辑分页和物理分页的区别是什么？

     - 逻辑分页,全部数据查到内存后再处理

     - 物理分页,直接使用limit关键字,在获取的时候就分页

129. mybatis 是否支持延迟加载？延迟加载的原理是什么？

     - 延迟加载即当sql为一对一/一对多关联时,关联数据只有在使用的时候才查出来
     - 支持,设置lazyLoadingEnabled=true 即可
     - 原理是返回的对象是cglib生成的子类,调用相关子对象时会被拦截,再去查询

130. 说一下 mybatis 的一级缓存和二级缓存？

     - 一级缓存基于SqlSession
     - 二级缓存基于namespace

131. mybatis 和 hibernate 的区别有哪些？

     - 

132. mybatis 有哪些执行器（Executor）？

     - **SimpleExecutor：**每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象
     - **ReuseExecutor：重复使用Statement对象
     - **BatchExecutor：**执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理。与JDBC批处理相同。

133. mybatis 分页插件的实现原理是什么？

     - PageHelper通过`ThreadLocal`来存放分页信息，从而可以做到在Service层实现无侵入性的Mybatis分页实现
     - [MyBatis之分页插件(PageHelper)工作原理](https://www.cnblogs.com/dengpengbo/p/10579631.html)

134. mybatis 如何编写一个自定义插件？

     - 

## 数据库/缓存

### - MySql

164. 数据库的三范式是什么？

     - 每一列属性都是不可再分的属性值，确保每一列的原子性
     - 需要确保数据库表中每一列都和主键相关，而不能只与主键的某一部分相关（主要针对联合主键而言）
     - 确保每列都和主键列直接相关，而不是间接相关
165. 一张自增表里面总共有 7  条数据，删除了最后 2  条数据，重启 mysql 数据库，又插入了一条数据，此时 id 是几？

     - 8
     - mysql自增id不会随着数据删除而改变
166. 如何获取当前数据库版本？

     - 在操作系统命令行,mysql -V
     - 在mysql内部,select version();
167. 说一下 ACID 是什么？

     - Atomicity原子性 
     - Consistency一致性
     - Isolation隔离性
     - Durability耐久性
168. char 和 varchar 的区别是什么？
169. float 和 double 的区别是什么？

     - float:浮点型，**含字节数为4**，32bit，数值范围为-3.4E38~3.4E38（7个有效位）

	- double:双精度实型，**含字节数为8**，64bit数值范围-1.7E308~1.7E308（15个有效位）

	- decimal:数字型，16字节，**128bit**，不存在精度损失（相对不存在，28个有效位后会报错），**常用于银行帐目计算**。（28个有效位）
170. mysql 的内连接、左连接、右连接有什么区别？
171. mysql 索引是怎么实现的？
     - B树
     - Hash
172. 怎么验证 mysql 的索引是否满足需求？
173. 说一下数据库的事务隔离？
174. 说一下 mysql 常用的引擎？
175. 说一下 mysql 的行锁和表锁？
176. 说一下乐观锁和悲观锁？
177. mysql 问题排查都有哪些手段？
178. 如何做 mysql 的性能优化？

### - Redis

179. redis 是什么？都有哪些使用场景？
180. redis 有哪些功能？
181. redis 和 memecache 有什么区别？
182. redis 为什么是单线程的？
183. 什么是缓存穿透？怎么解决？
184. redis 支持的数据类型有哪些？
185. redis 支持的 java 客户端都有哪些？
186. jedis 和 redisson 有哪些区别？
187. 怎么保证缓存和数据库数据的一致性？
188. redis 持久化有几种方式？
189. redis 怎么实现分布式锁？
190. redis 分布式锁有什么缺陷？
191. redis 如何做内存优化？
192. redis 淘汰策略有哪些？
193. redis 常见的性能问题有哪些？该如何解决？

## 扩展

### - 微服务

[微服务面试题](https://cloud.tencent.com/developer/article/1346868)

### - 设计模式

88. 说一下你熟悉的设计模式？
89. 简单工厂和抽象工厂有什么区别？

### - Git

90. [Git常用命令](git-learn.md)

### - Docker



### - 算法

91. 排序算法

92. 

    