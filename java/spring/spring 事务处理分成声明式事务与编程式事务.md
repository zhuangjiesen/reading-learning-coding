# spring 事务处理分成声明式事务与编程式事务
### 编程式事务就是在写代码过程中，在获取数据库连接session时，进行声明事务，接着对事务进行操作

### 声明式事务则是统一在配置文件中声明并配置事务类型，对于代码侵入少
### 声明式事务的实现是基于 spring aop ，将事务封装成 advisor 与 pointcut 的类型，对数据库(orm)的操作进行代理。

### 通过xml配置:
1. spring 文件中配置事务管理器
2. 配置事务代理计划，即事务隔离界别，事务回滚方式以及read-only 还有事务代理的方法名
3. 配置事务织入点与事务织入计划的映射，完成切面代理实现

```

	<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource"/>
	</bean>

	<!-- 事务相关控制配置：例如配置事务的传播机制 -->
	<tx:advice id="iccardTxAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="delete*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" no-rollback-for="java.lang.RuntimeException"/>
			<tx:method name="insert*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.RuntimeException" />
			<tx:method name="add*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.RuntimeException" />
			<tx:method name="create*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.RuntimeException" />
			<tx:method name="update*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" />

			<tx:method name="find*" propagation="SUPPORTS" />
			<tx:method name="get*" propagation="SUPPORTS" />
			<tx:method name="select*" propagation="REQUIRED"  />
			<tx:method name="query*" propagation="SUPPORTS" />
		</tx:attributes>
	</tx:advice>



	<aop:config>
		<aop:pointcut id="databaseStorePointcut" expression="execution(* com.java.*.service.*.*(..))"/>
		<aop:advisor pointcut-ref="databaseStorePointcut" advice-ref="iccardTxAdvice"/>
	</aop:config>


```

### 源码入口：

```
spring-tx.jar 包下，META-INF 包下的Resource Bundle 'spring' 中 配置tx标签的解析类(handlers文件中)
即 TxNamespaceHandler.java 类
每一个 advice 都对应生成了 TransactionInterceptor.class 的bean 
继承自 TransactionAspectSupport.class 在 TxNamespaceHandler 中将 <tx:advice> 标签属性都解析到
transactionAttributeSource 属性中，在动态代理的 TransactionInterceptor.class 的 invoke() 方法中拦截进行查看权限和判断方法，事务隔离级别的判断

主要进行动态代理操作事务的也是在 invoke() 方法中实现的
这里面针对了编程式事务和声明式事务进行了分别的处理，大致的处理都是一致的

通过ThreadLocal 存储当前线程的事务状态
1. 在 TransactionAspectSupport 中 createTransactionIfNecessary() 方法中 对 TransactionAspectSupport.TransactionInfo 设置了事务属性与信息
2.  createTransactionIfNecessary() 方法中还有一段代码  status = tm.getTransaction((TransactionDefinition)txAttr);  tm是 PlatformTransactionManager 类实现在 AbstractPlatformTransactionManager 中 getTransaction() 
有这一段  this.prepareSynchronization(status, (TransactionDefinition)definition); 通过对 TransactionSynchronizationManager 中进行对事务属性的操作

总之，都是通过对ThreadLocal的利用进行实现的

```

### 事务隔离级别：

1. Read Uncommitted（读取未提交内容）
       在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。
2. Read Committed（读取提交内容）
       这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别 也支持所谓的不可重复读（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。
3. Repeatable Read（可重读）
       这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和Falcon存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency Control）机制解决了该问题。
4. Serializable（可串行化） 
       这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。


### Spring事务的传播行为

1. REQUIRED:业务方法需要在一个容器里运行。如果方法运行时，已经处在一个事务中，那么加入到这个事务，否则自己新建一个新的事务。

2. NOT_SUPPORTED:声明方法不需要事务。如果方法没有关联到一个事务，容器不会为他开启事务，如果方法在一个事务中被调用，该事务会被挂起，调用结束后，原先的事务会恢复执行。

3. REQUIRESNEW:不管是否存在事务，该方法总汇为自己发起一个新的事务。如果方法已经运行在一个事务中，则原有事务挂起，新的事务被创建。

4. MANDATORY：该方法只能在一个已经存在的事务中执行，业务方法不能发起自己的事务。如果在没有事务的环境下被调用，容器抛出例外。

5. SUPPORTS:该方法在某个事务范围内被调用，则方法成为该事务的一部分。如果方法在该事务范围外被调用，该方法就在没有事务的环境下执行。

6. NEVER：该方法绝对不能在事务范围内执行。如果在就抛例外。只有该方法没有关联到任何事务，才正常执行。

7. NESTED:如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED属性执行。它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只对DataSourceTransactionManager事务管理器起效。



### 通过看代码解决的疑问是：
```
spring 的事务框架是spring 提供的，也就是说spring 事务框架的代理是在操作方法前通过设置ThreadLocal 属性 和操作方法完成或异常时从 ThreadLocal 中取出事务信息进行处理
而操作数据库的往往是通过orm数据库框架(mybatis / hibernate)进行处理的
所以通过查看源码发现
在 mybatis 源码中 SqlSessionFactoryBean类中 buildSqlSessionFactory() 方法中设置了默认的spring 的事务拦截器
也就是说orm 框架都会设置默认的事务处理器(SpringManagedTransactionFactory) 
在 事务处理器中 SpringManagedTransaction 的 openConnection() 
会找到通过获取数据库连接，然后设置ThreadLocal 属性 DataSourceUtils.doGetConnection() 方法中对 TransactionSynchronizationManager 的操作
完成与spring 事务框架的衔接，最后解开了我的疑问

spring 事务处理器，回滚条件并不是所有的异常都会导致事务回滚
有些异常是不会导致事务回滚的
关键在于

txInfo.transactiuonAttribute.rollbackOn(ex);

回滚条件

public boolean rollbackOn(Throwable ex) {
	return (ex instanceof RuntimeException || ex instanceof Error);
}

所以方法中抛出的异常需要 继承 RuntimeException 或者是 Error 类型，才能导致事务的回滚

```

