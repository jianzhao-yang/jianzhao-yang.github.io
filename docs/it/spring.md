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

Spring的IOC容器为我们完成了哪些工作? 或者说,我们使用容器的时候,它需要有哪些能力?

我们用容器,肯定是为了让它帮我们管理创建出来的对象,使用5w1h分析:

- why 为什么要交给容器进行管理? 为了更方便的进行对象管控,而不用我们手动管理
- what 要把什么交给它管理? 可以重复
- when
- where
- who
- how

基础能力: 

- 读取配置,以便知道哪些类需要放到容器中进行管理
- 
- 能够管理所需的类,能够按需创建bean,能够缓存bean

### 核心接口

#### BeanDefinition

> 用来描述bean的类,包含bean的class、是否单例、是否懒加载等信息

#### BeanDefinitionRegistry

> 用来盛放bean定义文件的注册表

#### BeanFactory

> 生产bean实例的工厂,因为要生产bean必须要有bean的元信息,所以也需要bean定义相关信息

#### SingletonBeanRegistry

> 因为单例bean全局只有一个,所以需要缓存起来重复使用

### Environment

> Environment就是程序启动时的外部环境,可以通过配置文件、启动参数等指定,主要包含: profile和properties,即项目启动环境和外部配置.

### ApplicationContext

#### PostProcessors


##### BeanFactoryPostProcessor

> 所有的bd已经全部加载完毕，然后可以对这些bd做一些属性的修改或者添加工作。

##### BeanDefinitionRegistryPostProcessor

> 该接口继承了BeanFactoryPostProcessor,所以是对它的一种扩展
>
> 所有常规bd已经加载完毕，然后可以再添加一些额外的bd。

官网的建议是BeanDefinitionRegistryPostProcessor用来添加**额外的**bd，而BeanFactoryPostProcessor用来修改bd。

##### BeanPostProcessor



#### Awares

BeanNameAware                   获得到容器中Bean的名称
BeanFactoryAware                获得当前bean Factory,从而调用容器的服务
ApplicationContextAware         当前的application context从而调用容器的服务
MessageSourceAware              得到message source从而得到文本信息
ApplicationEventPublisherAware  应用时间发布器,用于发布事件
ResourceLoaderAware             获取资源加载器,可以获得外部资源文件

##### Listeners

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

- xxxxxxxxxx  //Java 代码实现public class RadixSort implements IArraySort {    @Override    public int[] sort(int[] sourceArray) throws Exception {        // 对 arr 进行拷贝，不改变参数内容        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);        int maxDigit = getMaxDigit(arr);        return radixSort(arr, maxDigit);    }    /**     * 获取最高位数     */    private int getMaxDigit(int[] arr) {        int maxValue = getMaxValue(arr);        return getNumLenght(maxValue);    }    private int getMaxValue(int[] arr) {        int maxValue = arr[0];        for (int value : arr) {            if (maxValue < value) {                maxValue = value;            }        }        return maxValue;    }    protected int getNumLenght(long num) {        if (num == 0) {            return 1;        }        int lenght = 0;        for (long temp = num; temp != 0; temp /= 10) {            lenght++;        }        return lenght;    }    private int[] radixSort(int[] arr, int maxDigit) {        int mod = 10;        int dev = 1;        for (int i = 0; i < maxDigit; i++, dev *= 10, mod *= 10) {            // 考虑负数的情况，这里扩展一倍队列数，其中 [0-9]对应负数，[10-19]对应正数 (bucket + 10)            int[][] counter = new int[mod * 2][0];            for (int j = 0; j < arr.length; j++) {                int bucket = ((arr[j] % mod) / dev) + mod;                counter[bucket] = arrayAppend(counter[bucket], arr[j]);            }            int pos = 0;            for (int[] bucket : counter) {                for (int value : bucket) {                    arr[pos++] = value;                }            }        }        return arr;    }    private int[] arrayAppend(int[] arr, int value) {        arr = Arrays.copyOf(arr, arr.length + 1);        arr[arr.length - 1] = value;        return arr;    }}java
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

### 代理的几种方式

> 做代理要解决两个问题: 代理谁? 代理他们要做什么? 
>
> aspectj使用切面、连接点等概念很好的解决了代理谁的问题（代理谁√），但是静态代理不够灵活（怎么代理×）
>
> cglib的动态代理特性，以及相比jdk更高的性能，使得它能够更好的解决怎么代理的问题

#### AspectJ静态代理

> aspectj最大的好处就是定义了对代理类进行增强的类型和范围描述

PointcutParser

- parsePointcutExpression(expression) 就是解析描述表达式的入口
- couldMatchJoinPointsInType(clazz) 类是否被匹配到
- pointcutExpression.matchesMethodExecution(method).alwaysMatches()方法是否被匹配到



#### JDK动态代理

```java
// 定义要代理的类所使用的加载器、要继承的接口、以及处理方法
Proxy.newProxyInstance(this.getClass().getClassLoader(), interfaces, new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                // 可以在此添加各种处理
                Object invoke = method.invoke(args);
                
                return invoke;
    }
})
```



#### ASM动态代理

#### Cglib动态代理

> 从cglib可以看到,所谓代理,就是对于object.method([]Object args),这个最简单的类似main方法的前后处理增强

cglib的动态代理实现

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Foo.class);
enhancer.setInterfaces([]Interface.class);
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
                // 可以在此添加各种处理
                Object invoke = method.invoke(args);
                
                return invoke;
    }
});
```

### AspectJAwareAdvisorAutoProxyCreator



## Spring 事务

### 事务的传播行为

| 事务传播行为类型          |                   | 说明                                                         |
| ------------------------- | ----------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | required          | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。 |
| PROPAGATION_SUPPORTS      | supports          | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| PROPAGATION_MANDATORY     | mandatory（强制） | 使用当前的事务，如果当前没有事务，就抛出**异常**。           |
| PROPAGATION_REQUIRES_NEW  | requireds_new     | 新建事务，如果当前存在事务，把当前事务**挂起**。两个事务之间没有关联关系,互相不影响提交/回滚(新事务抛出异常被正确捕获的情况下). |
| PROPAGATION_NOT_SUPPORTED | not_supported     | 以非事务方式执行操作，如果当前存在事务，就把当前事务**挂起**。 |
| PROPAGATION_NEVER         | never             | 以非事务方式执行，如果当前存在事务，则抛出**异常**。         |
| PROPAGATION_NESTED        | nested            | 如果当前存在事务，则在**嵌套**事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。**子事务需要跟父事务一同提交,但是子事务可以独立回滚** |

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

### 事务可以解决重复插入的问题吗?

- 看隔离级别,读已提交级别下的select+insert/update甚至有可能加重重复插入问题.

