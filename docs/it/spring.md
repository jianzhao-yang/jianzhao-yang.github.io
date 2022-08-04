# Spring

## 整体流程

### SpringBoot启动流程

- 构造SpringApplication应用

  - 设置配置文件
  - 检测web环境
  - 创建应用监听器
  - 配置主类位置

- 启动 SpringApplication 应用

  - 加载应用启动监听模块 SpringApplicationRunListeners.starting()

  - 配置环境模块  prepareEnvironment() 加载配置文件 

    -  `ConfigFileApplicationListener.load()`

  - 应用上下文构建和准备  createApplicationContext() 、 prepareContext() 

- 自动配置

  - 准备refresh() 设置时间，校验参数等
  
  - beanFactory流程
  
    - obtainFreshBeanFactory() **创建** DefaultListableBeanFactory 
  
    - prepareBeanFactory() **配置**beanFactory，并添加一些默认的beanPostProcessor
  
    - postProcessBeanFactory() 由子类设置beanFactory创建后的**后置处理器**
  
    - invokeBeanFactoryPostProcessors() 执行这些系统的后置处理
  
      > - 找出beanFactory中所有的实现了BeanDefinitionRegistryPostProcessor接口和BeanFactoryPostProcessor接口的bean
      > - 对找出来的postProcessor进行排序
      > - 执行postProcessor中的postProcessBeanDefinitionRegistry()方法和postProcessBeanFactory()方法
  
    - registerBeanPostProcessors() 注册自定义的BeanPostProcessors
  
      > - *initMessageSource() 国际化*
      > - *initApplicationEventMulticaster()  初始化事件广播器*
      > - onRefresh() 子类实现refresh钩子
  
    - registerListeners() **注册监听器**
  
  - **finishBeanFactoryInitialization**() 初始化所有的单例bean
  
  - finishRefresh() 广播refresh完成事件

### Springboot配置文件优先级

​	按照从上到下的顺序加载，后加载的配置会覆盖先加载的

- 文件名优先级
  - application-xxx.properties
  - application.properties
- 文件路径优先级
  - classpath:/
  - classpath:/config/
  - file:./
  - file:./config/
- 文件类型优先级
  - properties
  - xml
  - yml
  - yaml

### Spring启动流程

### Spring Bean生命周期

#### 实例化 Instantiation

-  InstantiationAwareBeanPostProcessor  的before和after方法

#### 属性赋值 Populate

- 解决循环依赖问题

#### 初始化 Initialization

- Aware系列:  BeanNameAware、 BeanFactoryAware 、ApplicationContextAware 
-  BeanPostProcessor 的 beforeInitialization() 
-  InitializingBean 的 afterPropertiesSet ()
- 自定义初始化方法 PostProcess()
-  BeanPostProcessor 的 AfterInitialization() 

#### 使用

#### 销毁 Destruction

- DisposableBean

### BeanFactory和ApplicationContext

![](https://img2020.cnblogs.com/blog/1950875/202005/1950875-20200505112153634-1401928672.png)

- ApplicationContext是BeanFactory的子接口
- BeanFactory：bean工厂接口；负责创建bean实例；容器里面保存的所有单例bean其实是一个map；Spring最底层的接口
- ApplicationContext：是容器接口；更多负责容器功能的实现；（可以基于BeanFactory创建好的对象之上完成强大的容器）容器可以从map中获取这个bean，并且可以进行aop、di等操作

## IOC

### Spring 循环依赖

#### 三级缓存

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	/** 一级缓存,创建完毕的bean放在这里 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
	/** 三级缓存,bean被调用构造方法之后放在这里 */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
	/** 二级缓存,被循环依赖的bean会放在这里 */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
	/** 创建过程中的bean会放在这里记录 */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

#### 为什么要有第三级缓存?

- 三级缓存是为了判断循环依赖的时候，早期暴露出去已经被别人使用的 bean 和最终的 bean **是否是同一个 bean**，如果不是同一个则弹出异常，如果早期的对象没有被其他 bean 使用，而后期被修改了，不会产生异常，如果没有三级缓存，是无法判断是否有循环依赖，且早期的 bean 被循环依赖中的 bean 使用了。。
- 注:  BeanPostProcessor 可以修改bean的实际类型

### 依赖注入

#### 实现方式

- 属性注入方式
- 构造方法
- setter
- 方法注入

#### @Autowired和@Resource的区别

-  @Autowired 按照类型注入，如果需要按名称则需要与@Qualifier一起使用；@Resource先按照名字注入，找不到再按照类型注入
- @Autowired 可以作用在构造方法和入参上，@Resource能作用在类上
- @Autowired源自Spring，@Resource源自JDK

## AOP

- 

## Spring 事务

### 事务的传播行为

| 事务传播行为类型          |                   | 说明                                                         |
| ------------------------- | ----------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | required          | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。 |
| PROPAGATION_SUPPORTS      | supports          | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| PROPAGATION_MANDATORY     | mandatory（强制） | 使用当前的事务，如果当前没有事务，就抛出**异常**。           |
| PROPAGATION_REQUIRES_NEW  | requireds_new     | 新建事务，如果当前存在事务，把当前事务**挂起**。             |
| PROPAGATION_NOT_SUPPORTED | not_supported     | 以非事务方式执行操作，如果当前存在事务，就把当前事务**挂起**。 |
| PROPAGATION_NEVER         | never             | 以非事务方式执行，如果当前存在事务，则抛出**异常**。         |
| PROPAGATION_NESTED        | nested            | 如果当前存在事务，则在**嵌套**事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |

### 哪几种情况下事务会失效

- 非 public 修饰的方法上
- propagation 设置 PROPAGATION_SUPPORTS时无事务， PROPAGATION_NOT_SUPPORTED ， PROPAGATION_NEVER 
- rollbackFor 设置错误，非指定异常及其子类时不回滚
- 同一个类中方法调用，导致@Transactional失效。未进行特殊处理
- 异常被catch，未被抛出
- 数据库引擎不支持事务

### 同一个类中的调用怎么使用事务？

- 第一种：将b()方法抽出来，重新声明一个类，并且该类交由spring管理控制。
- 第二种：同时在a()上添加@Transactional注解或者在类上添加。
- 第三种：在原A类中的a()方法，改为 **((A)AopContext.currentProxy).b()**

## 

