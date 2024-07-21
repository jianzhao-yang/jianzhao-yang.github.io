## Mybatis

1. Mybatis核心概念

     ![](images\mybatis.jpg)

     - Executor是 Mybatis的内部执行器，它负责调用StatementHandler操作数据库，并把结果集通过ResultSetHandler进行自动映射，另外，他还处理了二级缓存的操作。从这里可以看出，我们也是可以通过插件来实现自定义的二级缓存的。
     - StatementHandler是Mybatis直接和数据库执行sql脚本的对象。另外它也实现了Mybatis的一级缓存。这里，我们可以使用插件来实现对一级缓存的操作(禁用等等)。
     - ParameterHandler是Mybatis实现Sql入参设置的对象。插件可以改变我们Sql的参数默认设置。
     - ResultSetHandler是Mybatis把ResultSet集合映射成POJO的接口对象。我们可以定义插件对Mybatis的结果集自动映射进行修改。

2. mybatis 中 #{}和 ${}的区别是什么？

     - #{}是预编译处理,${}是字符串替换

3. mybatis一二级缓存实现

   - 一级缓存是sqlSession级别的,在同一个sqlSession中,两次相同的查询第二次会使用第一次查询缓存的结果(在spring中,sqlsession是线程级别的)
   - 二级缓存是mapper级别的,同一mapper下的查询操作会缓存. 如果有其它服务/mapper同时操作数据库,会造成脏读(缓存和DB不一致)

4. xml转换sql过程 

   1.   Configuration类加载mapper
   2.  XMLMapperBuilder解析xml下的所有节点
   3.  XMLStatementBuilder 解析xml中的单个节点
   4.   XMLScriptBuilder 构建sql语句
   5.  MappedStatement表示的是XML中的一个SQL 

5. mapper和namespace的作用和区别

   - 在mybatis中，映射文件中的namespace是用于绑定Dao接口的，即面向接口编程。
   - 当你的namespace绑定接口后，你可以不用写接口实现类，mybatis会通过该绑定自动，帮你找到对应要执行的SQL语句。

6. mybatis的二级缓存失效是以什么为基本单位进行管理的?

   -  mybatis的[二级缓存](https://www.baidu.com/s?wd=二级缓存&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao)，进行增删改操时作会清空当前[命名空间](https://www.baidu.com/s?wd=命名空间&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao)中已缓存的数据。也就是说，缓存机制是以[命名空间](https://www.baidu.com/s?wd=命名空间&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao)为单位进行管理的。 

7. mybatis执行流程

   1. config文件解析

      -  ->SqlSessionFactoryBuilder.build()
      -  --使用 XMLConfigBuilder.parse()解析mybatis-config.xml，生成 Configuration

      -  <-返回带有Configuration属性的DefautSqlSessionFactory对象

   2.  使用SqlSessionFactory对象调用openSession获取SqlSession

   3.  SqlSession.getMapper 拿到Mapper对象的代理 

   4.  通过MapperProxy调用Maper中相应的方法

      - SqlSessioon创建executor,executor是对jdbc的封装,有simpleExecutor、batchExecutor、reuseExecutor等。 
      - executor构建StatementHandler，StatementHandler的作用
      - 使用ParameterHandler，对sql进行预处理;
      - 调用statement.executeXxx()执行sql；
      - 使用ResultSetHandler将数据库返回的结果集进行对象转换（ORM）；

8. 返回创建SqlSessionFactory对象
     2. 返回SqlSession的实现类DefaultSqlSession对象
     3. 返回一个MapperProxy的代理对象
     4.  执行CRUD流程。

9. mybatis 有几种分页方式？
     - limit
     - query(String userName, RowBounds rowBounds)
     - 实现拦截器
     - PageHelper

10. RowBounds 是一次性查询全部结果吗？为什么？

       - RowBounds是一次性查询全部结果
       - 从RowBounds源码看出，RowBounds最大数据量为Integer.MAX_VALUE(2147483647)

11. mybatis 逻辑分页和物理分页的区别是什么？

        - 逻辑分页,全部数据查到内存后再处理
        
        - 物理分页,直接使用limit关键字,在获取的时候就分页

12. mybatis 是否支持延迟加载？延迟加载的原理是什么？

        - 延迟加载即当sql为一对一/一对多关联时,关联数据只有在使用的时候才查出来
        - 支持,设置lazyLoadingEnabled=true 即可
        - 原理是返回的对象是cglib生成的子类,调用相关子对象时会被拦截,再去查询

13. 说一下 mybatis 的一级缓存和二级缓存？

        - 一级缓存基于SqlSession
        - 二级缓存基于namespace

14. mybatis 和 hibernate 的区别有哪些？

        1. hibernate是全自动，而mybatis是半自动（mybatis需要开发者编写sql语句，hibernate中封装了简单的增删改查可以不用编写操作数据库语句）
        
        2. hibernate数据库移植性远大于mybatis（由于mybatis中SQL语句是我们自己编写的，不同数据库的操作语句又不同所以使用mybatis时 更换数据库很麻烦，而hibernate会自动根据不同的数据库生成不同的操作语句）
        
        3. sql直接优化上，mybatis要比hibernate方便很多（mybatis是开发者编写的sql语句，优化上很方便，而hibernate是生成的sql无法直接优化，比较麻烦)

15. mybatis 有哪些执行器（Executor）？

        - SimpleExecutor：每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象
        - ReuseExecutor：重复使用Statement对象
        - BatchExecutor：执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理。与JDBC批处理相同。

16. mybatis 分页插件的实现原理是什么？

        - PageHelper通过`ThreadLocal`来存放分页信息，从而可以做到在Service层实现无侵入性的Mybatis分页实现
        - [MyBatis之分页插件(PageHelper)工作原理](https://www.cnblogs.com/dengpengbo/p/10579631.html)

17. mybatis 如何编写一个自定义插件？

        - 实现Interceptor接口的intercept方法，入参Invocation对象具体内容由类注解@Signature决定
        
        - ```java
          public @interface Signature {
              Class<?> type(); // 决定Invocation的target
              String method(); // 拦截什么类型的方法，如create、query
              Class<?>[] args();
          }
          public class Invocation {
              private final Object target;
              private final Method method;
              private final Object[] args;
          }
          ```
        
        - 只能拦截四种类型：
        
          - Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)          --执行sql
          - ParameterHandler (getParameterObject, setParameters)                     --获取、设置参数
          - ResultSetHandler (handleResultSets, handleOutputParameters)          --处理结果集
          - StatementHandler (prepare, parameterize, batch, update, query)          --记录sql

### 源码解析

#### 核心入口

> DefaultSqlSessionFactory的openSessionFromDataSource方法描述了mybatis执行所需的核心类:
>
> - Configuration mybatis 数据源相关配置,在初始化流程中解析加载
> - Executor 执行器,负责通过jdbc与db进行sql交互
> - SqlSession 负责一次连接生命周期的维护
> - Transaction 事务管理

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
---
        Environment environment = this.configuration.getEnvironment();
        TransactionFactory transactionFactory = this.getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        Executor executor = this.configuration.newExecutor(tx, execType);
        var8 = new DefaultSqlSession(this.configuration, executor, autoCommit);
---
    return var8;
}
```

#### 接口代理

> 参考文章: https://blog.csdn.net/fajing_feiyue/article/details/106174131

##### 包扫描

> 通过@MapperScan引入MapperScannerRegistrar类
>
> 将这些接口的实现指向了MapperFactoryBean进行代理

```java
@MapperScan(basePackages = "xxx.mapper")

@Import({MapperScannerRegistrar.class})
@Repeatable(MapperScans.class)
public @interface MapperScan {...}
```

> 将mapper加入spring bean定义扫描

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {
    void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry, String beanName) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
        --- 
 // 包扫描
    basePackages.addAll((Collection)Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText).collect(Collectors.toList()));
        --- 

            builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));
        // 将MapperScannerConfigurer注册到spring
        registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
    }
```

> MapperScannerConfigurer【核心】将注册所有mapper的bean定义

```java
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 加载mybatis配置
        if (this.processPropertyPlaceHolders) {
            this.processPropertyPlaceHolders();
        }
		// ClassPathMapperScanner extends ClassPathBeanDefinitionScanner
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
---
        scanner.registerFilters();
        // 这里将扫描mapper包下所有类
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ",; \t\n"));
    }
```

> ClassPathBeanDefinitionScanner用于bean扫描,属于spring类

```java
public int scan(String... basePackages) {
   int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
  // 调用子类ClassPathMapperScanner扫描包
   doScan(basePackages);

   // Register annotation config processors, if necessary.
   if (this.includeAnnotationConfig) {
      AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }

   return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

```

> ClassPathMapperScanner 核心处理类，扫描mapper包下所有接口，将这些接口的实现指向MapperFactoryBean进行代理

```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 扫描所有bean定义
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
                    + "' package. Please check your configuration.");
    } else {
        // 处理bean定义
        processBeanDefinitions(beanDefinitions);
    }
}
    
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        String beanClassName = definition.getBeanClassName();
---
        definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
        // 这里标注了该bean定义是哪个接口的实现类
        //【核心】即将这些接口的实现指向了MapperFactoryBean进行代理
        definition.setBeanClass(this.mapperFactoryBeanClass);
---
}
```



##### 代理实现

> mybatis mapper动态代理实现类所在包为: org.apache.ibatis.binding; 核心类如下:
>
> - MapperMethod 对应Mapper中的每一个方法
> - MapperProxy 对应一个接口的代理类
> - MapperProxyFactory 用于生成所需mapper代理类
> - MapperRegistry 映射注册类,用于管理(解析/保存)所有Mapper代理类



##### Configuration类

Configuration类是mybatis中最核心的配置类,里面保存了所有的配置,包括:mybatis-config.xml的全局配置、mapper.xml中的mapper配置、与SQL对应的statement,以及Mapper接口、代理类和他们的映射关系。核心变量包括：

- MapperRegistry：保存Mapper接口和Mapper代理类的映射关系，以及Mapper代理类的具体实现
- InterceptorChain： 存放所有插件
- Map<String, MappedStatement> ： 存放所有statement（select、insert、update、delete语句）
- 