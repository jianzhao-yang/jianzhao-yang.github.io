# 分布式系统常见面试题

## SpringCloud

### SpringBoot

1. SpringBoot有哪些优点？
   - 内置servlet容器（tomcat、jetty等）
   - 简化默认配置（约定大于配置）
   - 简化maven依赖（starter）
   - 自动配置 autoconfiguration
   - yml、properties配置文件，dev、prd自动选择配置
   - 自动装配bean
   - 命令行cli
   - 监视actuator 
2. 如何使用热启动（修改代码自动重启）？
   - 引入spring-boot-devtools 
3. springboot常用的starter有哪些 ？
   - spring-boot-starter-web 嵌入tomcat和web开发需要servlet与jsp支持 
   - spring-boot-starter-data-jpa 数据库支持 
   - spring-boot-starter-data-redis redis数据库支持 
   - spring-boot-starter-data-solr solr支持 
   - mybatis-spring-boot-starter 第三方的mybatis集成starter
4. springboot自动配置的原理 
   在spring程序main方法中 添加@SpringBootApplication或者@EnableAutoConfiguration 
   会自动去maven中读取每个starter中的spring.factories文件 该文件里配置了所有需要被创建spring容器中的bean
5. 常见的starter会包几个方面的内容？分别是什么？
   - 常见的starter会包括下面四个方面的内容 
   - 自动配置文件，根据classpath是否存在指定的类来决定是否要执行该功能的自动配置。 
   - spring.factories，非常重要，指导Spring Boot找到指定的自动配置文件。 
   - endpoint：可以理解为一个admin，包含对服务的描述、界面、交互(业务信息的查询)。 
   - health indicator:该starter提供的服务的健康指标。
6. 自定义springboot-starter注意事项？
   - springboot默认scan的包名是其main类所在的包名。如果引入的starter包名不一样，需要自己添加scan。 @ComponentScan(basePackages = {"com.xixicat.demo","com.xixicat.sms"}) 
   -  对于starter中有feign的，需要额外指定 @EnableFeignClients(basePackages = {"com.xixicat.sms"})
   -  对于exclude一些autoConfig @EnableAutoConfiguration(exclude ={MetricFilterAutoConfiguration.class})
7. 常见Conditional*注解？
   - @ConditionalOnBean
   - @ConditionalOnClass
   - @ConditionalOnMissingBean
   - @ConditionalOnMissingClass
   - @ConditionalOnWebApplication
   - @ConditionalOnResource
   - @ConditionalOnProperty

### Zuul 网关

### Ribbon 负载均衡

### Eureka 服务发现

### Feign 服务调用

### Hystrix 服务熔断

### Apollo 配置中心

## MQ

### RabbitMQ

### RocketMQ

## Zookeeper

## Dubbo