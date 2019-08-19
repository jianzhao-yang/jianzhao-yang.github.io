## Java面试个人总结

### 1.java 基础

· 集合
· 线程

[线程的状态](https://dl.iteye.com/upload/picture/pic/116719/7e76cc17-0ad5-3ff3-954e-1f83463519d1.jpg): new、runnable、running、blocked、waiting/timed-waiting、dead

[sleep()与wait()区别](https://blog.csdn.net/linfanhehe/article/details/78737685)： 

sleep会自动唤醒，wait不会；

sleep不会释放锁，wait会；

wait只能在同步方法块中使用，sleep可以在任何地方使用；

sleep属于Thread类，wait属于Object类；

[锁升级：无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁](https://www.liangzl.com/get-article-detail-135712.html)

[死锁](https://www.cnblogs.com/bopo/p/9228834.html)

[四种引用类型](https://www.cnblogs.com/liyutian/p/9690974.html)

[synchronized底层实现原理](https://baijiahao.baidu.com/s?id=1612142459503895416&wfr=spider&for=pc)

· 并发
· 锁
· 反射
· I/O
· Java8新特性
· JVM
· GC

### 2.网络

· HTTP
· TCP/IP
· NIO/AIO

### 3.数据库

· 数据库设计三范式
· 事务的传播和隔离
· Mysql
· SQL
· 事务
· 索引
· 优化
· 分库分表

### 4.常用框架

· Spring
· SpringMVC
· SpringBoot
· Mybatis

### 5.设计模式

### 6.缓存

 · redis

### 7.微服务

### 8.工具

· GIT
· docker

### 9.分布式

· zookeeper
· dubbo
· spring cloud