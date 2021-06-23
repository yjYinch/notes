# Spring事务

最近，项目中使用@Transacional注解导致事务隔离问题，问题大致情况如下：

```java
public class Controller{
    
    public Result methodA(){
        // 1.调用service层的方法
        a();
    }
}

public class Service{
    @Transactional(rollbackfor = Exception.class)
    public Result a(){
        // for循环，批量操作数据库，只要有一条失败，就回滚全部的数据
        for(){
            // 2.操作数据库
        }
        // 3.操完完成之后，通知设备
        callCpe(map);
    }
}

public class CPE(){
    public void callCpe(Map map){
        
        // 多线程执行
        // 4.然后涉及到数据库的操作
    }
}
```

以上会出现问题是，当2.for循环数据库已经操作完成，第4步设计到更新表操作，会出现找不到更新不到数据的情况。原因是因为callCpe()跟a()不在同一个事务中。

**解决方法：**

1、将4步多线程操作改为单线程，这种方式不合适，因为发送请求给cpe是多线程操作

2、将a()方法里的 callCpe(map)，提取到controller层，这种方式修改起来最简单，能够在事务完全提交的情况下去调用 callCpe(map)。

3、移除声明式编程@Transactional注解，改为编程式注解，当手动提交完成后，去调用 callCpe(map)。但是这种方式对代码量改动较大，因此最后采取第2种方式。



### 事务概念

**什么是事务？**

> 事务是正确执行一系列操作，使得数据库从一种状态转换为另一种状态，且保证操作全部成功，或者全部失败。

**事务的原则？**

事务需满足**ACID**原则，分别是**原子性**、**一致性**、**独立性**、**持久性**。



Spring事务分为**编程式**和**声明式**两种。

* **编程式事务**：是按照编写代码的方式来完成事务提交、回滚等操作
* **声明式事务**：基于AOP，将业务代码与事务声明分开。比如@Transactional 注解
* **编程式事务**：手动提交或者回滚数据。



## 一、声明式事务

<font color = green>@Transactional注解参数说明</font>

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

   /**
    * 事务管理器
    */
   @AliasFor("transactionManager")
   String value() default "";

   /**
    * 事务管理器，与value互为别名
    */   
   @AliasFor("value")
   String transactionManager() default "";

   /**
    * 事务管理器的标签
    */  
   String[] label() default {};

   /**
    * 事务传播行为设置，默认是Propagation.REQUIRED
    */
   Propagation propagation() default Propagation.REQUIRED;

   /**
    * 事务隔离级别，默认是Isolation.DEFAULT
    */   
   Isolation isolation() default Isolation.DEFAULT;

   /**
    * 事务超时设置
    */  
   int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

   
   String timeoutString() default "";

   /**
    * 读写或只读事务，默认读写
    */   
   boolean readOnly() default false;

   /**
    * 异常回滚，指定具体的类，如果不填，默认为非受检异常
    */   
   Class<? extends Throwable>[] rollbackFor() default {};

   /**
    * 异常回滚，按照异常类的名字
    */     
   String[] rollbackForClassName() default {};

   /**
    * 对于指定的异常不回滚
    */   
   Class<? extends Throwable>[] noRollbackFor() default {};

   /**
    * 异常不回滚，按照异常类的名字
    */       
   String[] noRollbackForClassName() default {};

}
```

<font color = green>**事务管理器：transactionManager**</font>

| 数据访问技术 | 实现                         |
| ------------ | ---------------------------- |
| JDBC         | DatasourceTransactionManager |
| JPA          | JpaTransactionManager        |
| Hibernate    | HibernateTransactionManager  |

<font color = green>**事务的隔离级别：isolation**</font>

| 隔离级别         | 隔离级别的值 | 说明                                                         |
| ---------------- | ------------ | ------------------------------------------------------------ |
| Read Uncommitted | 0            | 事务中的修改，即使没有提交，对其它事务也是可见的。会出现**脏读** |
| Read Committed   | 1            | 一个事务从开始到提交之前，所做的任何事情都是对其它事务不可见的。**避免了脏读**，但会导致**不可重复读**（两次读取执行同样的查询，结果不一致）。 |
| Repeatable Read  | 2            | 同一事务中，多次读取同样的记录结果是一样的。**避免了脏读，允许重复读**，但会出现**幻读**（当某个事务再次读取该范围的记录时，另外个事务再在该范围插入一条记录，当之前的事务再次读取该范围的记录时，会产生幻行） |
| Serializable     | 3            | 强制事务串行执行，避免了幻读的问题，它会在读取的每一行数据都加锁，所以可能会导致大量的等待和锁竞争问题，因此效率低，不建议使用。 |

<font color = green>**隔离级别特点**</font>

| 隔离级别         | 脏读可能性 | 不可重复读可能性 | 幻读可能性 | 加锁读 |
| ---------------- | ---------- | ---------------- | ---------- | ------ |
| Read Uncommitted | Yes        | Yes              | Yes        | No     |
| Read Committed   | No         | Yes              | Yes        | No     |
| Repeatable Read  | No         | No               | Yes        | No     |
| Serializable     | No         | No               | No         | Yes    |

MySQL默认的隔离级别是Repeatable Read。



<font color = green>**事务的传播行为**</font>

* **TransactionDefinition.PROPAGATION_REQUIRED**：**@Transactional的默认值**，如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
* **TransactionDefinition.PROPAGATION_REQUIRES_NEW**：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
* **TransactionDefinition.PROPAGATION_SUPPORTS**：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
* **TransactionDefinition.PROPAGATION_NOT_SUPPORTED**：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
* **TransactionDefinition.PROPAGATION_NEVER**：以非事务方式运行，如果当前存在事务，则抛出异常。
* **TransactionDefinition.PROPAGATION_MANDATORY**：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
* **TransactionDefinition.PROPAGATION_NESTED**：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行，如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。



### @Transactional注解失效的场景

<font color=red>场景1：@Transactional注解的不是public方法</font>

比如，如下场景，虽然方法a()不会报错，但是事务无效。

```java
public class Test{
    @Transactional
    protected void a(){
        //do something
    }
}
```

**解决方法：**被标注的方法修饰符改为public即可。



<font color=red>场景2：同一类中的其它没有`@Transactional`注解的方法调用带有注解的方法时，有`@Transactional`注解的方法的事务被忽略，不会发生回滚，即@Transactional注解失效。</font>

比如：

```java
public class Test{
    
    public void a(){
        //调用方法b
        b();
    }
    
    @Transactional
    public void b(){
        //do something
    }
}
```

**解决方法：**在方法a()上面也加入@Transactional注解，让其都在同一个事务中。



<font  color=red>场景3：事务方法里开启了新线程去执行了其它事务方法，其它事务方法的事务不受当前事务方法控制。</font>

比如：方法a()内部开启多线程调用方法b()，当方法b()抛出异常进行回滚时，方法a()不会回滚。

```java
public class Test{
    
    @Transactional(rollbackFor = Exception.class)
    public void a(){
        new Thread(()-> {
            try {
                b();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }
    
    @Transactional(rollbackFor = Exception.class)
    public void b() throws Exception {
        try {
            // do something
        } catch (RuntimeException e){
            throw new Exception();
        }
    }
}
```

原因：Spring 是通过TransactionSynchronizationManager类中的ThreadLocal变量来获取事务中的数据库连接`Connection`的，因此，不同的线程`Connection`不一样，即不会在同一个事务中。

**解决方案：**若想两者都在同一事务中，则方法里面应当禁用多线程。



<font  color=red>场景4：方法里的异常被catch，并没有向上throw</font>

```java
public class Test{
    @Transactional
    public void a(){
        try{
            // do something
        }catch(Exception e){
            log.error(".....")
        }
    }
}
```

**解决方案：**原则上来说，用@Transactional注解标注的方法里面对于数据库的操作不应该有try catch，如果进行try catch的话，应该手动抛出异常。



<font  color=red>场景5：方法里抛出的异常为受检异常，且在@Transactional注解没有指定异常类型</font>

例如：当@Transactional注解是默认异常时，当出现异常时，@Transactional注解将会失效。

```java
public class Test{
    @Transactional
    public void a() throws Exception{
        try{
            // do something
        }catch(Exception e){
            log.error(".....");
            throw new Exception();
        }
    }
}
```

**原因**：默认情况下，Spring只有在抛出异常为unchecked非受检异常才会出现回滚事务， 而抛出 checked 异常则不会导致事务回滚。因此需要通过rollback手动设置异常类型。

**解决办法**：在@Transactional(rollbackFor = Exception.class)指定异常为非受检和受检类型，此时事务将会回滚。

![异常结构](https://gitee.com/zhangyijun0229/picture-for-pic-go/raw/master/img/20210623141108.png)



<font  color=red>场景6：数据库引擎不支持事务</font>

这种场景出现的很少，由于现在MySQL默认的存储引擎为InnoDB，它是支持事务的，对于不支持事务的引擎，现在很少遇到。

### 原理分析

spring 在扫描bean的时候会扫描方法上是否包含@Transactional注解，如果包含，spring会为这个bean动态地生成一个子类（即代理类，proxy），代理类是继承原来那个bean的。此时，当这个有注解的方法被调用的时候，实际上是由代理类来调用的，代理类在调用之前就会启动transaction。然而，如果这个有注解的方法是被同一个类中的其他方法调用的，那么该方法的调用并没有通过代理类，而是直接通过原来的那个bean，所以就不会启动transaction，我们看到的现象就是@Transactional注解无效。

案例分析，以上述**场景2**为例，下面场景就会出现方法B()的注解失效。

**代理原因分析：**

```java
@Service
public class Test{
  
    public method a(){   //标记1 准备调用方法a()，但是由于Test类里面有方法b()被注解@Transactional标注，因此Test类就创建了代理类。
        b();
    }
    
    @Transactinal
    public method b(){ //标记3
    }
}
 
//Spring扫描注解后，创建了另外一个代理类，并为有注解的方法插入一个startTransaction()方法：
class proxy$Test{
    Test t = new Test();
   
    method a(){    //标记2
        t.a();    //由于a()没有注解，所以不会启动transaction，而是直接调用A的实例的a()方法
    }
    
     method b(){    
        startTransaction();
        t.b();
    }
}
```

当调用class Test的方法a()时，会被代理拦截，调用proxy$A的a()，然后调用t.a()，并不会开启事务。



**源码分析**

下图展示了调用一个带有@Transactional注解的方法流程图。

![@Transactional注解执行流程](https://gitee.com/zhangyijun0229/picture-for-pic-go/raw/master/img/20210622173931.png)



举例：

```java
@Service
public class A{
    @Transactional()
    public void a(){
        //do something
    }
}
```

当调用方法a()时，每个带有@Transactional注解的方法都会创建一个切面,所有的事务处理逻辑就是由这个切面完成的,这个@Transactional注解会对执行方法前后进行拦截处理。

1. CglibAopProxy类的私有静态内部类DynamicAdvisedInterceptor的intercept会对其进行拦截，然后调用CglibMethodInvocation对象的proceed()方法，继续向上交给父类ReflectiveMethodInvocation的invoke()方法，其父类将其交给实现了MethodInterceptor实现类中的invoke()方法TransactionInterceptor类里。
2. 然后事务拦截器TransactionInterceptor会调用invoke()方法，然后调用父类TransactionAspectSupport会调用invokeWithinTransaction()方法。
3. invokeWithinTransaction()方法首先会开启事务startTransaction，然后调用a()方法。
4. 如果a()方法处理异常，则调用completeTransactionAfterThrowing()方法，对事务进行回滚。
5. 如果a()方法处理结果正常，则调用commitTransactionAfterReturning()方法，对事务进行提交。
6. 上面的completeTransactionAfterThrowing和commitTransactionAfterReturning方法最终也是使用的jdbc的Savepoint和Connection来完成回滚和提交,就如同我们使用原生的jdbc代码那样操作。

TransactionInterceptor

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
    @Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// 调用父类的方法
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
			@Override
			@Nullable
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
			@Override
			public Object getTarget() {
				return invocation.getThis();
			}
			@Override
			public Object[] getArguments() {
				return invocation.getArguments();
			}
		});
	}
}
```

TransactionAspectSupport

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
    @Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {
        
        // 省略大量无关代码...
        
        
        // 这里TransactionAttributeSource对象保存了应用内所有方法上的@Transactional注解属性信息,利用Map来保存,其中key由目标方法
        // method对象+目标类targetClass对象组成,
        // 所以通过method+targetClass就能唯一找到目标方法上的@Transactional注解性信息
		TransactionAttributeSource tas = getTransactionAttributeSource();
        
        // 这个TransactionAttribute对象就保存了目标方法@Transactional注解所有的属性配置,如timeout,propagation,readOnly等等,
        // 后续就利用这些属性完成对应的操作
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
        
        // ......
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

		Object retVal;
		try {
			// 这里最终会调用到目标方法
			retVal = invocation.proceedWithInvocation();
		} catch (Throwable ex) {
			// 目标方法抛出了异常,根据@Transactional注解属性配置决定是否需要回滚事务
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
        // ......
        // 目标方法正常执行完成,提交事务
		commitTransactionAfterReturning(txInfo);
		return retVal;
        }
}
```



通过以上可知，在使用@Transactional注解的时候，有许多地方需要考虑全面，一不小心，代码可能就出现问题。如果你不想使用@Transactional注解，那么编程式事务就为你提供了另外一种解决方案。



## 二、编程式事务

在spring出现之前，以前操作数据库的时候，需要首先**开启事务**，然后**操作数据**，接着**commit/rollback**，最后**关闭数据库连接**，这就是编程式事务的核心。下面就开始介绍一下在Spring中如何使用编程式事务。

### Spring事务API分析

**事务管理：**按照给定的事务规则来执行提交或者回滚操作。

这句话可以分为3个层面来解释

1、给定的事务规则

2、按照....执行操作

3、事物的状态，提交还是回滚

在Spring中，分为对这3个部分做了接口定义。

1、给定的事务规则 （TransactionDefinition）

2、按照....执行操作 （PlatformTransactionManager）

3、事物的状态 （TransactionStatus）



**TransactionDefinition接口，主要定义了事务的传播行为和事务的隔离级别。**

```java

public interface TransactionDefinition {
    // 事务的传播行为
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    
    // 事务的隔离级别
    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED = 2;
    int ISOLATION_REPEATABLE_READ = 4;
    int ISOLATION_SERIALIZABLE = 8;
    int TIMEOUT_DEFAULT = -1;

    default int getPropagationBehavior() {
        return 0;
    }

    default int getIsolationLevel() {
        return -1;
    }

    default int getTimeout() {
        return -1;
    }

    default boolean isReadOnly() {
        return false;
    }

    @Nullable
    default String getName() {
        return null;
    }

    static TransactionDefinition withDefaults() {
        return StaticTransactionDefinition.INSTANCE;
    }
}
```



**PlatformTransactionManager：事务管理**

```java
public interface PlatformTransactionManager extends TransactionManager {
	/**
	 * 根据指定的传播行为，返回当前活动的事务或创建一个新事务
	 */
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;
			
	/**
	 * 提交
	 */
	void commit(TransactionStatus status) throws TransactionException;

	/**
	 * 回滚
	 */
	void rollback(TransactionStatus status) throws TransactionException;

}
```



**TransactionStatus：表示事务的状态**

```java
public interface TransactionStatus {
	/**
	 * 是否是新事物
	 */
    boolean isNewTransaction();
    
    /**
	 * 是否为只回滚
	 */
    boolean isRollbackOnly();
    
    /**
	 * 是否已完成
	 */
    boolean isCompleted();
    /**
	 * 是否有保存点
	 */
    boolean hasSavepoint();
}
```



### 编程式事务实现方式

**事务管理器（PlatformTransactionManager）方式**

操作步骤：

​	1、获取事务管理器PlatformTransactionManager

​	2、创建事务管理器属性对象TransactionDefinition

​	3、获取事务状态对象TransactionStatus

​	4、数据库操作

​	5、执行commit或rollback操作

代码示例：

```java
@Service
public class TransactionalService {
    private static final Logger log = LoggerFactory.getLogger(TestController.class);

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private TransactionDefinition transactionDefinition;

    @Autowired
    private RoleMapper roleMapper;

    public Integer addRoleByManager(List<Role> roles){
        // 获取事务状态
        TransactionStatus transactionStatus = transactionManager.getTransaction(transactionDefinition);
        Integer result;
        try {
            result = roleMapper.insert(roles);
            transactionManager.commit(transactionStatus);
            log.info("新增role成功，role信息：{}", roles);
        } catch (Exception e) {
            log.error("新增role失败，原因：{}", e.getMessage());
            transactionManager.rollback(transactionStatus);
            result = 0;
        }
        return result;
    }
}
```

**模板事务（TransactionTemplate）的方式（官方推荐√）**

操作步骤：

​	1、获取TransactionTemplate对象

​	2、选择任务返回结果类型

​	3、执行任务

可以看出这样做的好处是不用手动提交或者回滚事务，当第3步执行任务异常时，会帮我们自动回滚，因为它指定的异常为Throwable。

代码示例：

```java
@Service
public class TransactionalService {
    private static final Logger log = LoggerFactory.getLogger(TestController.class);
    
    /**
     * 1.获取TransactionTemplate对象
     */ 
    @Autowired
    private TransactionTemplate transactionTemplate;

    @Autowired
    private RoleMapper roleMapper;

    public Integer addRole(List<Role> roles) {
        // 2.选择任务返回结果类型为Integer
        return transactionTemplate.execute(new TransactionCallback<Integer>() {
            @Override
            public Integer doInTransaction(TransactionStatus status) {
                Integer result;
                try {
                    // 3.执行任务
                    result = roleMapper.insert(roles);
                    log.info("新增role成功，role信息：{}", roles);
                } catch (Exception e) {
                    log.error("新增role失败，原因：{}", e.getMessage());
                    status.setRollbackOnly();
                    result = 0;
                }
                return result;
            }
        });
    }
}
```
