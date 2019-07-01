[Java代理模式以及责任链模式和Spring AOP原理](https://www.imooc.com/video/15692)   
[Spring IOC原理](https://blog.csdn.net/it_man/article/details/4402245)    
[IOC与DI](http://blog.xiaohansong.com/2015/10/21/IoC-and-DI/)  
[FactoryBean原理](https://blog.csdn.net/u013185616/article/details/52335864)

## 常用配置和话题
>* Scope
>* Spring EL: @Value
>* 初始化和销毁：@Bean的initMethod和destroyMethod | JSR-250的@PostConstruct和@PreDestroy
>* @Profile:为不同环境下使用不同的配置提供了支持
>* 事件(Application Event)
>* Spring Aware: 使得bean能够获得Spring容器提供的服务
>* 任务执行器(TaskExecutor)
>* 计划任务(@Scheduled)
>* @Conditiional:根据满足某一条件创建一个bean
>* @Enable*的工作原理：使用@import导入配置类


### Spring MVC
1. 理解MVC和三层架构的关系（展示层(MVC)+应用层+数据访问层）   


## Spring Boot创建Beans的过程分析|启动流程分析
SpringApplication的run方法：
```text
public ConfigurableApplicationContext run(String... args) {
  //主要这三步
  context = createApplicationContext();
  prepareContext(context, enviroment, listeners, applicationArguments, printedBanner, printedBanner);
  refreshContext(context);
}
```
createApplicationContext中判断是否为web环境创建相应的context的Class对象，然后调用BeanUtils.initiate(Class<T>)方法，返回Context对象，以AnnotationConfigEmbeddedWebApplicationContext为例。其
父类GenericApplicationContext的无参构造器中有如下代码：
```text
this.beanFactory = new DefaultListableBeanFactory();
```  
这个beanFactory是生成beans对象的工厂类。GenericApplicationContext的父类AbstractAutowireCapableBeanFactory中有两个方法：
```text
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
  //调用的各个方法中比如invokeAwareMethods方法里调用了Aware系列的接口方法。
}
createBean(...) {
...
}
这两个方法在后文会有提及。  
```
进入prepareContext方法：
```text
private void prepareContext(...) {
  //只写出核心步骤
  postProcessApplicationContext(context);
  applyInitializers(context);
  load(context, sources.toArray(new Object[sources.size()]));
}
```
1. postProcessApplicationContext(context)方法：
```text
postProcessApplicationContext(context) {
  //向bean工厂添加一个beannamegenerator，
  //设置ResourcecLoader和ClassLoader
}
```
2. applyInitializers(context)方法：
```text
applyInitializers(context) {
  //所有的上下文初始化器对ApplicationContext进行自定义
  for(ApplicationContextInitializer initializer: getInitializers) {
    ...
    initializer.initialize(context);
  }
}
```
3. load(context, sources.toArray(new Object[] sources) :
```text
//这个方法用来加载各种beans到context对象中，sources代表各种资源对象，BeanDefinitionLoader内部提供了方法读取解析这些资源对象中的beans。
protected void load(ApplicationContext context, Object[] sources) {
  BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
  loader.setBeanNameGenerator(this.beanNameGenerator);
  loader.setResourceLoader(this.resourceLoader);
  loader.setEnviroment(this.enviroment);
  loader.load();
}
```
接下来就是refreshContext方法，这个方法内部调用了refresh(context)方法，该方法调用了(AbstractApplicationContext)applicationContext.refresh()方法:
```text
public void refresh() throws BeansException, IllegalStateException {
  //各种初始化工作
  synchronized(this.startupShutdownMonitor) {
    prepareRefresh();
    ConfgurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    prepareBeanFactory(beanFactory);
    try {
      postProcessBeanFactory(beanFactory);
      invokeBeanFactoryPostProcessors(beanFactory);
      registerBeanPostProcessors(beanFactory);
      initMessageSource();
      initApplicationEventMulticaster();
      onRefresh();
      registerListeners();
      /*这里进行了非懒加载bean的初始化工作，该方法内部调用了beanFactory.preInstantiateSingletons()方法，这个beanFactory是DefaultListableBeanFactory的实例对象，进入其中：
      preInstantiateSingletons() {
        ...
        //getBean中调用了AbstractBeanFactory的doGetBean方法，其中又调用了createBean方法(与BeanDefinition相关)
        getBean(beanName)
      }   
      */
      finishBeanFactoryInitialization(beanFactory);
      finishRefresh();
    }
  }
}
```

##: Spring事务
逻辑上的一组操作，要么都执行，要么都不执行。  
事务的特性：
>* 原子性
>* 一致性
>* 隔离性：并发访问数据库时，一个用户的事务不被其他事务干扰
>* 持久性
 
Spring并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate等持久化机制所提供的相关框架的事务来实现。
```text
//对于不同的持久层框架，Spring提供了具体的接口实现类。  
public interface PlatformTransactionManager {
  TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
  void commit(TransactionStatus status) throws TransactionException;
  void rollback(TransactionStatus status) throws TransactionException;
}
```
Spring中事务属性通过TransactionDefinition和TransactionStatus体现。  
### TransactionDefinition接口实现如下：
```text
public interface TransactionDefinition {
  /*
  ps:
  不可重复读：一个事务内两次读取数据前后不一致；
  幻读：一个事务内第二次读取到了第一次读取时不存在的数据。
  五种隔离级别：
  ISOLATION_DEFAULT：使用后端数据库默认的隔离级别，MySql默认REPEATABLE_READ隔离级别，Oracle默认为READ_COMMITED；
  ISOLATION_READ_UNCOMMITED：读未提交，导致脏读、幻读和不可重复读；
  ISOLATION_READ_COMMITED: 读已提交，防止脏读，不能防止幻读和不可重复读；
  ISOLATION_REPEATABLE_READ：防止脏读和不可重复读，但是幻读仍可能发生；
  ISOLATION_SERIALIZATION: 完全服从ACID的隔离级别，所有事务依次逐个执行，互不干扰。
  七种传播行为：
  PROPAGATION_REQUIRED: 如果当前存在事务，则加入该事务；如果当前不存在事务，则创建一个事务；
  PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前不存在事务，则以非事务的方式继续运行；
  PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前不存在事务，则抛出异常；
  以上三种支持当前事务。  
  PROPAGATION_REQUIRES_NEW: 创建一个新的事务，如果当前存在事务，则把当前事务挂起；
  PROPAGATION_NOT_SUPPORTED: 以非事务方式运行，如果当前存在事务，则把当前事务挂起；
  PROPAGATION_NEVER: 以非事务方式运行，如果当前存在事务，则抛出异常；
  
  PROPAGATION_NESTED: 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果没有当前事务，等价于PROPAGATION_REQUIRED。
  
  ps: 注意自调用情况导致propagation不生效，与Spring AOP原理有关。
  */
  
  //传播行为：当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。
  int getPropagationBehavior();
  //隔离级别
  int getIsolationLevel();
  String getName();
  //超时，事务所允许执行的最长时间
  int getTimeout();
  //对事务资源(数据源、JMS资源等)的读写权限
  boolean isReadOnly();
}

```
### TransactionStatus接口介绍
```text
这个接口用来记录事务的状态：
public interface TransactionStatus {
  boolean isNewTransaction();
  boolean hasSavepoint();
  void setRollbackOnly(); //设为只回滚
  boolean isRollbackOnly();
  boolean isCompleted;
}
```






