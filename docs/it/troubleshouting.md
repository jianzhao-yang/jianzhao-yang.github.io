## 实战问题记录

### 一次数据库连接异常导致的系统崩溃

- 场景： 我之前有维护过一个后台管理项目，后来应公司需求需要将其迁移到内部云上，之前因为项目比较小所以项目代码和数据库都在同一台服务器上，迁移之后变成了集群 + 云数据库；迁移之后项目每过一段时间就会崩溃，具体表现为服务器不响应任何请求，但是重启之后又恢复正常。
- 分析： 根据这种表现，可以判断很可能时哪里出现了阻塞导致的。于是就考虑将堆栈信息dump下来，在本地进行分析。首先是根据jstack -l pid> dump.log 将信息导出，然后查看其中每个线程执行的位置，有没有Deadlock、waiting、blocked等状态，果然发现了有一个线程是处于waiting的状态的。仔细分析这个栈信息，发现是一个获取数据库连接的操作。之后找到对应的数据库连接池管理相关类，发现这个项目当时使用的是自己写的一个连接池（旧版本的dbcp bug会导致数据库连接爆满、c3p0也有死锁bug），而在获取数据库连接的时候并没有超时操作，又因为迁移之后项目和数据库分离了，走的是共用的内网环境，所以有可能会出现网络不稳定导致获取连接失败的情况。
- 解决： 将原来的获取连接的操作变为一个子线程操作，并使用Future定时获取该操作的返回结果，如果一段时间没有获取成功，就返回错误信息/重试；或者更换为druid连接池

### ThreadLocal导致的OOM

- 场景： 项目中每过一段时间（几天）就会发生OOM
- 分析： 因为项目启动时使用了-XX：+HeapDumpOnOutOfMemoryError参数，所以会自动dump堆信息到文件heap.hprof；此时我们只需要使用相关工具分析dump文件即可。将生成的dump文件导出，使用jvisualvm进行查看，发现有很多订单对象存在于内存中，吃掉了大量的内存。于是对系统进行排查，，发现由于需求变更，很多方法签名不能满足正常使用，之前传的参数不够用，所以使用了threadlocal进行跨方法的对象传递，但是在使用了之后却没有进行及时的清理。导致出现内存占用过高的情况
- 解决： 排查threadlocal对象使用的地方，及时使用remove进行清理

### JVM调优记录

https://www.cnblogs.com/itworkers/p/11791778.html

- jstat

  - 使用 jstat -gcutil pid 毫秒时间 查看gc情况
  - 使用 jstat -gccause pid 毫秒时间 查看gc原因

- jinfo

  - 使用 jinfo -flags pid 查看jvm启动参数

- 查看项目运行时间

  - ps -p <pid> -o etime 

- jvm参数

  - ```
    JAVA_OPTS="-Xms2g -Xmx2g -Xmn512m -XX:MaxPermSize=256m  -server -Xss256k -XX:PermSize=128M -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/log/gclog/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/log/jvmdump/jvm.bin -XX:+UseConcMarkSweepGC -XX:+UseParNewGC  -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:+CMSClassUnloadingEnabled -XX:+TieredCompilation  -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime -XX:+PrintHeapAtGC
    ```

- jmap

  - jmap -histo:live pid 查看存活对象
  -  jmap -heap $pid，查看进程的 堆状态
  -  jmap -dump:live,format=b,file=/tmp/application.hprof pid  dump堆文件

- jstack

  - jstack -l pid 查看栈信息

### 一次http头大小设置错误导致的OOM

之前项目中有一次要上线一个活动，需要进行压测，结果压测环境出现了OOM。由于我在JVM启动里设置了OOM后dump文件，所以就可以拿到dump文件进行分析。我是使用mat进行分析的，分析过程中发现有一个byte[]数组非常多，占用了很多内存，每个数组大小都有10M左右。查看所有对这些数组的引用，发现很多都指向tomcat的一个ReadBuffer类。因为正常情况下tomcat是不会出现这种问题的，所以排查是否是自己设置的参数错误导致的。最后排查到是因为项目中权限校验使用的是jwt的token形式，之前因为这个导致header的大小超过了默认的4k，所以修改了设置server.max-http-header-size: 10485760 #10M，从网上复制过来的配置。将默认的最大header设置为10M，但是没有想到tomcat分配是 MaxHttpHeaderSizeReadBuffer + ReadBuffer ，直接以最大值进行设置，就导致一个http请求占用的内存过大，当进行压测时，http请求数激增，会导致OOM

