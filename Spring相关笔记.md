# Spring 容器启动流程

下面所有的代码都是以`ClassPathXmlApplicationContext`容器为例进行源码的分析。

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    //递归初始化父容器
    super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
    //容器启动入口
		refresh();
	}
}
```

**AbstractApplicationContext.refresh()**

```java
// Prepare this context for refreshing.
prepareRefresh();
// 初始化容器工厂
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
// 添加内置处理
// 配置需要忽略注入的类
// 配置特定化的注入规则，有些依赖注入不需要我们自己配置，Spring会自动为我们配置注入的Bean
// 注册默认Bean
prepareBeanFactory(beanFactory);
try {
	// 调用BeanFactory的后置处理，这里应该是指容器自己注入的处理器
	postProcessBeanFactory(beanFactory);

    // 首先触发包括BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry
    // 然后触发BeanDefinitionRegistryPostProcessor的postProcessBeanFactory
    // 最后触发BeanFactoryPostProcessors的postProcessBeanFactory
    // 此时这些注册的Processor还没有进行初始化，所以会调用getBean方法进行初始化
	invokeBeanFactoryPostProcessors(beanFactory);

	// 注册顺序：
    // PriorityOrdered
    // Ordered
    // regular
    // internal（MergedBeanDefinitionPostProcessor）重复注册
	registerBeanPostProcessors(beanFactory);

	// 初始化本地化信息
	initMessageSource();

	// 初始化事件广播器
    // 未定义使用SimpleApplicationEventMulticaster
	initApplicationEventMulticaster();

	// Initialize other special beans in specific context subclasses.
	onRefresh();

	// Add beans that implement ApplicationListener as listeners
    // 将监听器注册到广播器中
    // 广播早期事件
    // 触发SmartInstantiationAwareBeanPostProcessor.predictBeanType()
	registerListeners();

	// 对非懒加载的Bean进行初始化
    // Initialize LoadTimeWeaverAware beans early to allow for registering their
    // transformers early.
	finishBeanFactoryInitialization(beanFactory);

	// Last step: publish corresponding event.
	finishRefresh();
}catch (BeansException ex) {
    // Destroy already created singletons to avoid dangling resources.
	destroyBeans();

	// Reset 'active' flag.
	cancelRefresh(ex);

	// Propagate exception to caller.
	throw ex;
}finally {
	// Reset common introspection caches in Spring's core, since we
	// might not ever need metadata for singleton beans anymore...
	resetCommonCaches();
}
```

**AbstractRefreshableApplicationContext.refreshBeanFactory()**

```java
//obtainFreshBeanFactory调用获取BeanFactory
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
      destroyBeans();
	  closeBeanFactory();
  }
  try {
    //创建容器
    DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
    	//读取配置信息生成BeanDefinitions的map信息
    	//注册之后会Reset all bean definition caches for the given bean
    	//触发MergedBeanDefinitionPostProcessor
    	//触发时机在Reset all bean definitions that have the given bean as parent之前
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

**DefaultListableBeanFactory.preInstantiateSingletons()**

finishBeanFactoryInitialization调用。

该方法开始执行非懒加载Bean的初始化，这里会触发一个`SmartInitializingSingleton`回调，该接口定义了 afterSingletonsInstantiated()方法，该方法会在非懒加载的Bean初始化后进行调用。

**AbstractBeanFactory.doGetBean()**

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

  final String beanName = transformedBeanName(name);
	Object bean;

	// 查看是否存在当前Bean实例缓存
    // 这里会涉及到三级缓存用于处理循环依赖
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
    //Get the object for the given bean instance, either the bean
	 	//instance itself or its created object in case of a FactoryBean.
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
    //对于非单例Bean无法处理循环依赖，直接报错
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
    	// 在当前容器中寻找相应的Bean，找不到递归在父容器中寻找
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
      		//获取相应的BeanDefinition
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
      		// 递归调用直到没有任何依赖
            // 获取depends-on配置的依赖
            // @AutoWired注解是通过BeanPostProcessor实现依赖注入的
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException();
					}
					registerDependentBean(dep, beanName);
					try {
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException();
					}
				}
			}

			// Create bean instance.
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
        		// 原型模式，指的是每次调用时，会重新创建该类的一个实例
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException();
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException();
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException();
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			throw new BeanNotOfRequiredTypeException();
		}
	}
	return (T) bean;
}


protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //一级缓存此时的对象已经实例化和初始化完成
    Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
            // 二级缓存和三级缓存相关联
            // 三级缓存是一个工厂，二级缓存时从该工厂中获取到的Bean
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
                    //获取三级缓存
					singletonObject = singletonFactory.getObject();
                    //设置二级缓存
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

**AbstractAutowireCapableBeanFactory.createBean()**

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)throws BeanCreationException {
  RootBeanDefinition mbdToUse = mbd;
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException();
	}

	try {
    	//如果存在InstantiationAwareBeanPostProcessor
    	//返回实现类的执行结果
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException();
	}

	try {
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isTraceEnabled()) {
			logger.trace("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException();
	}
}
```

**AbstractAutowireCapableBeanFactory.doCreateBean()**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
    	//实例化Bean
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
        		//触发MergedBeanDefinitionPostProcessors.postProcessMergedBeanDefinition()
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException();
			}
			mbd.postProcessed = true;
		}
	}

	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
        // 设置三级缓存
        // 此时的Bean只是实例化并没有初始化
        // 提供一个特殊的工厂可以获取给定的Bean
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
    	//依赖注入
        //resolveValueIfNecessary-》resolveReference
		populateBean(beanName, mbd, instanceWrapper);
    	//触发BeanPostProcessor
    	//触发InitializingBean
    	//触发制定的init-method
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException();
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException();
				}
			}
		}
	}

	// Register bean as disposable.
	try {
    	//判断Bean是否实现了disposable接口，或者制定了相应的销毁方法
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException();
	}

	return exposedObject;
}
```

# Spring AOP

## 实现方法

* ##### 经典的基于代理的AOP实现

  ```xml
  <!-- 定义被代理者 -->
  <bean id="demo" class="com.springAOP.bean.demo"></bean>
     
  <!-- 定义通知内容，也就是切入点执行前后需要做的事情 -->
  <!-- spring定义了不同的通知接口 -->
  <bean id="advise" class="com.springAOP.bean.advise"></bean>
     
  <!-- 定义切入点位置 -->
  <bean id="Pointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">
  	<property name="pattern" value=""></property>
  </bean>
     
  <!-- 使切入点与通知相关联，完成切面配置 -->
  <bean id="Advisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
  	<property name="advice" ref="advise"></property>   	
     	<property name="pointcut" ref="Pointcut"></property>
  </bean>
     
  <!-- 设置代理 -->
  //通过这种方式配置代理，通过getBean从容器中获取的应该时该代理Bean，而不是被代理Bean
  <bean id="proxy" class="org.springframework.aop.framework.ProxyFactoryBean">
  	<!-- 代理的对象 -->
  	<property name="target" ref="demo"></property>
  	<!-- 使用切面 -->
  	<property name="interceptorNames" value="Advisor"></property>
  	<!-- 代理接口 -->
      //没有实现接口会怎么样？？？
  	<property name="proxyInterfaces" value=""></property> 
  </bean>
  ```

  ```xml
  <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
  ```

  可以用上面的配置代码代替上面的代理配置实现自动创建代理，spring为自动创建代理提供了以下几种方式：

  * `BeanNameAutoProxyCreator`

    为一组特定配置名的Bean自动创建代理实例的代理创建器

  * `DefaultAdvisorAutoProxyCreator`

    它会对容器中所有的Advisor进行扫描，自动将这些切面应用到匹配的Bean中

  * `AnnotationAwareAspectJAutoProxyCreator`

    为包含AspectJ注解的Bean自动创建代理实例

  上面这些自动代理类都实现了BeanPostProcessor

* **基于AspectJ注解配置**

  利用BeanPostProcessor实现

  ```xml
  <aop:aspectj-autoproxy />
  ```

* **基于aop:config配置**

  ```xml
  <aop:config>
      //切点定义
  	<aop:pointcut id="Pointcut" expression="切点表达式" />
      //指定通知类，并不需要实现什么接口
  	<aop:aspect ref="helloAspect">
      	<!—以下使用了两种方法定义切入点  pointcut-ref和pointcut-->
          <aop:before pointcut-ref="Pointcut" method="beforeAdvice" />
          <aop:after pointcut="切点表达式" method="afterFinallyAdvice" />
      </aop:aspect>
  </aop:config>
  ```

## 代理逻辑

* **经典的基于代理的AOP实现**

  ```java
  Object sharedInstance = getSingleton(beanName);
  	if (sharedInstance != null && args == null) {
  		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  	}
  ```

  上面代码中的`getObjectForBeanInstance`就是触发ProxyFactory的地方，此时Spring容器的初始化已经结束，用户调用getBean方法获取指定的Bean，由于已经存在Bean实例，所以这里的sharedInstance非空（这里就是和初始化区分的地方）。getObjectForBeanInstance方法还会经过层层调用，主要就是判断是否为FactoryBean（ProxyBean是一个FactoryBean）并调用FactoryBean的getObject方法根据配置信息返回代理对象。

# Spring事务

## 事务的特性

* 原子性 
* 一致性 
* 隔离性
* 持久性 

## Spring事物管理接口

* PlatformTransactionManager //执行事务操作

  Spring并不直接管理事务，而是提供了多种事务管理器 ，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

  ```java
  Public interface PlatformTransactionManager(){ 
  
      TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
  
      Void commit(TransactionStatus status) throws TransactionException; 
  
      Void rollback(TransactionStatus status) throws TransactionException; 
  
  } 
  ```

* TransactionDefinition //描述事务规则

  ```java
  public interface TransactionDefinition { 
  
        // 返回事务的传播行为 
        int getPropagationBehavior(); 
        // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据 
        int getIsolationLevel(); 
        String getName()； 
        int getTimeout(); 
        // 返回是否优化为只读事务。 
        boolean isReadOnly(); 
  } 
  ```

  TransactionDefinition 接口中定义了五个表示隔离级别的常量： 

  * TransactionDefinition.ISOLATION_DEFAULT//使用后端数据库默认隔离级别
  * TransactionDefinition.ISOLATION_READ_UNCOMMITTED 
  * TransactionDefinition.ISOLATION_READ_COMMITTED 
  * TransactionDefinition.ISOLATION_READ_COMMITTED 
  * TransactionDefinition.ISOLATION_SERIALIZABLE 

  TransactionDefinition 接口中定义以下几个表示事务传播行为的常量： 

  * TransactionDefinition.PROPAGATION_REQUIRED

    存在事务就在该事务中运行，不存在启动新事务

  * TransactionDefinition.PROPAGATION_SUPPORTS 

    不存在以非事务运行，存在就在当前事务中运行？？

  * TransactionDefinition.PROPAGATION_MANDATORY 

    必须在事务中运行，不存在抛异常

  * TransactionDefinition.PROPAGATION_REQUIRES_NEW 

    当前方法必须在自己的事务中，会启动新事务，挂起当前事务

  * TransactionDefinition.PROPAGATION_NOT_SUPPORTED

    存在就挂起当前事务，以非事务运行

  * TransactionDefinition.PROPAGATION_NEVER 

    存在事务，抛异常

  * TransactionDefinition.PROPAGATION_NESTED 

    嵌套事物

    内嵌事务并不是一个独立的事务，它依赖于外部事务的存在，只有通过外部的事务提交，才能引起内部事务的提交，嵌套的子事务不能单独提交。对于嵌入式的事务的处理，内嵌的事务异常并不会引起外部事务的回滚。内部事务如果出错只会标记回滚标识不会执行回滚操作，外部事物在提交时会进行判断，如果事物链上存在回滚表示那么会直接进行回滚不会提交。

  在TransactionDefinition中还能定义事物的回滚规则，默认情况下Spring中的事务处理只对RuntimeException和Error异常进行回滚 ，但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。

  ```xml
  @Transactional(propagation=Propagation.REQUIRED,rollbackFor=Exception.class)
  ```

* TransactionStatus //可以看作事物本身

  ```java
  public interface TransactionStatus{
      boolean isNewTransaction(); // 是否是新的事物
      boolean hasSavepoint(); // 是否有恢复点
      void setRollbackOnly();  // 设置为只回滚
      boolean isRollbackOnly(); // 是否为只回滚
      boolean isCompleted; // 是否已完成
  } 
  ```

## Spring编程式事务管理

编程式的事物管理主要就是利用上面描述的三个借口通过编码的形式在程序中进行人为的事物管理。

```java
public class DemoServiceImpl implements DemoService {
    private XXX demoMethod;
    private TransactionDefinition txDefinition;
    private PlatformTransactionManager txManager;
    //开启事物
  	TransactionStatus txStatus = txManager.getTransaction(txDefinition);
    try {
      demoMethod.process();//需要事物管理的方法执行
      txManager.commit(txStatus);//事务提交
    } catch (Exception e) {
      txManager.rollback(txStatus);//事务回滚
    }
}
```

配置文件如下：

```xml
<bean id="DemoService" class="DemoServiceImpl">
    <property name="demoMethod" ref="demoMethod"/>
    <property name="txManager" ref="transactionManager"/>
    <property name="txDefinition">
    	<bean class="org.springframework.transaction.support.DefaultTransactionDefinition">
        //定义事物的传播行为
    		<property name="propagationBehaviorName" value="PROPAGATION_REQUIRED"/>
    	</bean>
    </property>
</bean>
```

如果嫌上面的操作太过繁琐，可以利用基于TransactionTemplate 的编程式事务管理，这里不赘述，可网查。

## Spring声明式事务管理

Spring 的声明式事务管理是建立在 Spring AOP 机制之上的，其本质是对目标方法前后进行拦截，并在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

```xml
<!-- 配置 TransactionManager -->
<bean id="txManager"
   //class表示具体的PlatformTransactionManager实现
   class="org.springframework.orm.hibernate3.HibernateTransactionManager">
  	//注入数据源
    <property name="sessionFactory" ref="sessionFactory" />
</bean>

<!-- 配置事务增强处理的切入点，以保证其被恰当的织入 -->    
<aop:config>
    <!-- 切点 -->
    <aop:pointcut expression="切点表达式" id="pointnName"/>
    <!-- 声明式事务的切入 -->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="pointName" />
</aop:config>

<!-- 由txAdvice切面定义事务增强处理 -->
//相当于配置TransactionDefinition信息
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <!-- get打头的方法为只读方法,因此将read-only设为 true -->
        <tx:method name="get*" read-only="true" />
        <!-- 其他方法为读写方法,因此将read-only设为 false -->
        <tx:method name="*" read-only="false" propagation="REQUIRED"
            isolation="DEFAULT" />
    </tx:attributes>
</tx:advice>
```

除了上面描述的基于xml的事物配置方法，还能采用基于@Transactional注解的声明式事务管理，在需要进行事物管理的方法上加上注解，并提供相应的事物描述信息，然后在xml配置文件中不需要进行aop声明，主需要添加下面的配置信息。

```xml
<tx:annotation-driven transaction-manager="transactionManager"/>
```

## 动态代理

Spring使用AOP来实现声明式事务，动态代理是Spring实现AOP的默认方法，主要分为两种：

* JDK动态代理

  JDK动态代理面向接口，通过反射生成目标代理接口的匿名实现类。

  ```java
  //loader:类加载器
  //interfaces：需要代理的接口
  //handler：代理处理方法，实现了InvocationHandler接口
  Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces, InvocationHandler handler)
  ```

* CGLIB动态代理

  CGLIB 是一个基于ASM的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成，CGLIB通过继承方式实现代理 ， 如果方法使用了final修饰或者是私有方法，是不能被增强的 。
  
  ```java
  Enhancer enhancer = new Enhancer();
  //设置被代理类
  enhancer.setSuperclass(HelloConcrete.class);
  //设置拦截方法，实现MethodInterceptor接口的intercept方法
  enhancer.setCallback(new MyMethodInterceptor());
  HelloConcrete hello = (HelloConcrete)enhancer.create();
  System.out.println(hello.sayHello("I love you!"));
  ```

事务拦截器的继承关系图如下：

![img](https://pic1.zhimg.com/80/v2-e1b27f9e03b55e4695b72177d3fc60d8_hd.png)

## Spring声明式事务处理源码分析

声明式事务处理通过 tx:annotationdriven标签进行配置，具体看翻看上面的配置信息。在TxNamespaceHandler类的init方法会为该标签注册解析器用于处理该标签。

```java
public void init(){
 registerBeanDefinitionParser("annotationdriven",newAnnotationDrivenBeanDefinitionParser());
}
```

AnnotationDrivenBeanDefinitionParser.parse()用来解析该事务标签，该方法会根据该标签的mode属性来判断AOP的处理方式，如果为aspectj会调用registerTransactionAspect进行处理，否则调用AopAutoProxyConfigurer.configureAutoProxyCreator配置自动代理。该类中主要创建了三个Bean：

* TransactionAttributeSourceAnnotation	//TransactionAttributeSource.class类型
* TransactionInterceptor                               //TransactionInterceptor.class类型
* TransactionAttributeSourceAdvisor         //BeanFactoryTransactionAttributeSourceAdvisor.class类型

其中TransactionAttributeSource被注入TransactionAttributeSourceAdvisor的transactionAttributeSource属性中，TransactionInterceptor被注入TransactionAttributeSourceAdvisor的adviceBeanName属性中。

TransactionAttributeSourceAdvisor是advise的实现类，所以在找相应的增强器时也会找到该类，并和其他增强器一样被织入代理中。所以当代理被调用时会调用该增强类，而该增强类中注入了TransactionInterceptor事务处理器，所以在调用事务增强器增强的代理类时会首先执行TransactionInterceptor进行增强。

AopAutoProxyConfigurer.configureAutoProxyCreator方法中会调用一下方法：

```java
AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext,element);
```

该方法的主要目的是注册InfrastructureAdvisorAutoProxyCreator类型的bean，InfrastructureAdvisorAutoProxyCreator间接实现了SmartInstantiationAwareBeanPostProcessor，而SmartInstantiationAwareBeanPostProcessor又继承自InstantiationAwareBeanPostProcessor，也就是说在Spring中，所有bean实例化时Spring都会保证调用其postProcessAfterInitialization方法。postProcessAfterInitialization方法会调用wrapIfNecessary函数，该函数主要包含以下两种功能：

* 找出指定Bean对应的增强器
* 根据找出的增强器创建代理

### TransactionInterceptor

事务拦截器支撑着整个事务架构，用于向需要事务操作的方法注入事务相关的处理逻辑。该类继承了MethodInterceptor，所以当该拦截器被调用时会调用它的invoke方法。invoke方法首先根据事物属性获取对应的TransactionManager，然后根据以下判断条件判断事物处理的方式，进行不同的事务操作。

```java
//第一步：
//获取事务属性
//根据事物属性获取相应的TransactionManager
//第二步：
//编程式的事务处理是没有事务属性的
if(txAttr==null||!(tminstanceofCallbackPreferringPlatformTransactionManager)){
  //声明式事务处理
}else{
  //编程式事务处理
}
```

### 声明式事务处理

```java
//创建事务
TransactionInfo txInfo = createTransactionIfNecessary(tm,txAttr,joinpointIdentification);
Object retVal=null;
try{
  //执行被增强方法
  retVal=invocation.proceed();
}catch(Throwable ex){
  //异常回滚
  completeTransactionAfterThrowing(txInfo,ex);
  //这里对捕获的异常进行了外抛，表示如果当前事务包含在另一个事务内，则会导致外部事物回滚
  throw ex;
}finally{
  //清除信息
  cleanupTransactionInfo(txInfo);
}
commitTransactionAfterReturning(txInfo);
return retVal; 
```

### 创建事务createTransactionIfNecessary

```java
protected TransactionInfo createTransactionIfNecessary(PlatformTransactionManager tm,TransactionAttribut etxAttr,final String joinpointIdentification{
  //如果没有名称指定则使用方法唯一标识，并使用DelegatingTransactionAttribute封装txAttr
  if(txAttr!=null&&txAttr.getName()==null){
    txAttr=new DelegatingTransactionAttribute(txAttr){
      @Override
      public String getName(){
        return joinpointIdentification;
      }
    };
  }
  TransactionStatus status=null;
  if(txAttr!=null){
    if(tm!=null){
      //获取Transaction,开启事务
      Statusstatus=tm.getTransaction(txAttr);
    }else{
      if(logger.isDebugEnabled()){
        logger.debug("");
      }
    }
  }
  //根据指定的属性与status准备一个TransactionInfo
  //后面事务的回滚和提交都利用该信息
  return prepareTransactionInfo(tm,txAttr,joinpointIdentification,status);
}
```

**getTransaction()**

```java
public final TransactionStatus getTransaction(TransactionDefinition definition)throws TransactionException{
  //获取事务
  //设置保存点开关
  Object transaction=doGetTransaction();
  boolean debugEnabled=logger.isDebugEnabled();
  if(definition==null){
    //根据传播行为的不同，进行不同的事务创建处理
    definition=new DefaultTransactionDefinition();
  }
  //判断当前线程是否存在事务，判读依据为当前线程记录的连接不为空且连接中(connectionHolder)中的
  //transactionActive属性不为空
  if(isExistingTransaction(transaction)){
    //当前线程已经存在事务
    //根据当前事务的传播方式进行不同的事务启动方式
    return handleExistingTransaction(definition,transaction,debugEnabled);
  }
  //事务超时设置验证
  if(definition.getTimeout()<TransactionDefinition.TIMEOUT_DEFAULT){
    throw new InvalidTimeoutException("Invalidtransactiontimeout",definition.getTimeout());
  }
  //如果当前线程不存在事务，但是propagationBehavior却被声明为PROPAGATION_MANDATORY抛出异常	
  if(definition.getPropagationBehavior()==TransactionDefinition.PROPAGATION_MANDATORY){
  	throw new IllegalTransactionStateException("");
	}else if(definition.getPropagationBehavior()==TransactionDefinition.PROPAGATION_REQUIRED
       ||definition.getPropagationBehavior()==TransactionDefinition.PROPAGATION_REQUIRES_NEW
       ||definition.getPropagationBehavior()==TransactionDefinition.PROPAGATION_NESTED){
  	//PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED都需要新建事务
  	//空挂起
  	SuspendedResourcesHolder suspendedResources=suspend(null);
  	if(debugEnabled){
   	 logger.debug();
  	}
  	try{
    	boolean newSynchronization=(getTransactionSynchronization()!=SYNCHRONIZATION_NEVER);
    	DefaultTransactionStatus status=new TransactionStatus(
      	definition,transaction,true,newSynchronization,debugEnabled,suspendedResources);
    	//构造transaction,包括设置ConnectionHolder、隔离级别、timout
      //更改自动提交设置，事务提交由Spring控制
      //如果是新连接，绑定到当前线程
    	doBegin(transaction,definition);
    	//新同步事务的设置，针对于当前线程的设置
    	prepareSynchronization(status,definition);
    	return status;
  	}catch(RuntimeException ex){
      resume(null,suspendedResources);
    	throw ex;
  	}catch(Error err){
    	resume(null,suspendedResources);
    	throw err;
  	}
	}else{
    boolean new Synchronization=(getTransactionSynchronization()==SYNCHRONIZATION_ALWAYS);
    return prepareTransactionStatus(
      definition,null,true,newSynchronization,debugEnabled,null);
  }
} 
```

### 回滚事务cleanupTransactionInfo

```java
protected void completeTransactionAfterThrowing(TransactionInfo txInfo,Throwable ex){
  //当抛出异常时首先判断当前是否存在事务，这是基础依据
  if(txInfo!=null&&txInfo.hasTransaction()){
    //这里判断是否回滚默认的依据是抛出的异常是否是RuntimeException或者是Error的类型
    //默认只对这两种异常进行回滚，但是我们可以通过配置进行拓展
    //@Transactional(propagation=Propagation.REQUIRED,rollbackFor=Exception.class)
    if(txInfo.transactionAttribute.rollbackOn(ex)){
      //这里对于回滚时的异常进行了捕获并向外抛，同样会影响外部事物
      try{
        //根据TransactionStatus信息进行回滚处理
        //触发回滚前后的监听函数
        //如果有保存点回会回退到保存点
        //独立新事务直接回退
        //如果当前事务不是独立的事务，那么只能标记状态，等到事务链执行完毕后统一回滚
        //清空记录的资源并将挂起的资源恢复
        txInfo.getTransactionManager().rollback(txInfo.GetTransactionStatus());
      }catch(TransactionSystemException ex2){
        ex2.initApplicationException(ex);
        throw ex2;
      }catch(RuntimeException ex2){
        throw ex2;
      }catch(Error err){
        throw err;
      }
    }else{
      //如果不满足回滚条件即使抛出异常也同样会提交
      //这里对于提交时的异常进行了捕获并向外抛，同样会影响外部事物
      //提交时出现异常回回滚
      try{
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
      }catch(TransactionSystemException ex2){
        ex2.initApplicationException(ex);
        throw ex2;
      }catch(RuntimeException ex2){
        throw ex2;
      }catch(Error err){
        throw err;
      }
    }
  }
}
```

### 事务提交commitTransactionAfterReturning

commitTransactionAfterReturning会调用下面的方法进行事务的提交操作。

```java
public final void commit(TransactionStatus status)throws TransactionException{
  //如果事务已完成抛异常
  if(status.isCompleted()){
    throw new IllegalTransactionStateException();
  }
  DefaultTransactionStatus defStatus=(DefaultTransactionStatus)status;
  //如果在事务链中已经被标记回滚，那么不会尝试提交事务，直接回滚
  //嵌套事务
  if(defStatus.isLocalRollbackOnly()){
    processRollback(defStatus);
    return;
  }
  if(!shouldCommitOnGlobalRollbackOnly()&&defStatus.isGlobalRollbackOnly()){
    processRollback(defStatus);
    if(status.isNewTransaction()||isFailEarlyOnGlobalRollbackOnly()){
      throw new UnexpectedRollbackException();
    }
    return;
  }
  //处理事务提交
  //触发相应的监听器
  //如果存在保存点不进行提交，并清除保存点信息
  //不是新事务不提交
  processCommit(defStatus);
}
```

对于内嵌事务，在Spring中正常的处理方式是将内嵌事务开始之前设置保存点，一旦内嵌事务出现异常便根据保存点信息进行回滚，但是如果没有出现异常，内嵌事务并不会单独提交，而是根据事务流由最外层事务负责提交，所以如果当前存在保存点信息便不是最外层事务，不做保存操作。

当某个嵌入事务发生回滚的时候会设置回滚标识，而等到外部事务提交时，一旦判断出当前事务流被设置了回滚标识，则由外部事务来统一进行整体事务的回滚。

# Spring Bean初始化干预

 Spring 允许在 Bean 在初始化完成后以及 Bean 销毁前执行特定的操作，常用的设定方式有以下三种： 

* 通过实现 `InitializingBean`/`DisposableBean` 接口来定制初始化之后/销毁之前的操作方法；
* 通过 <bean> 元素的 `init-method`/`destroy-method` 属性指定初始化之后 /销毁之前调用的操作方法；
* 在指定方法上加上`@PostConstruct` 或`@PreDestroy`注解来制定该方法是在初始化之后还是销毁之前调用，根据调试显示该方法还是通过`BeanPostProcess`接口实现的，存在多个相同注解时执行顺序根据代码中的声明顺序。
* 实现`BeanPostProcess`接口的`postProcessBeforeInitialization`和`postProcessAfterInitialization`方法，如果存在多个该接口实现类执行顺序根据配置文件的声明顺序。

Bean初始化和销毁执行顺序：

* 构造方法（属于实例化阶段）
* `BeanPostProcessor`的`postProcessBeforeInitialization`方法
* 类中添加了注解`@PostConstruct` 的方法
* `InitializingBean`的`afterPropertiesSet`方法
* bean的指定的初始化方法： `init-method`
* `BeanPostProcessor`的`postProcessAftrInitialization`方法
* 类中添加了注解`@PreDestroy`的方法
* `DisposableBean` 的destroy方法
* bean的指定的初始化方法： `destroy-method`

# Spring PostProcessor

* `BeanDefinitionRegistryPostProcessor`

  触发时机：refresh.invokeBeanFactoryPostProcessors()

  定义了`postProcessBeanDefinitionRegistry`方法，可以让我们实现自定义的注册Bean定义的逻辑，是一种特殊的`BeanFactoryPostProcessor`。

* `BeanFactoryPostProcessor`

  触发时机：refresh.invokeBeanFactoryPostProcessors()

  定义了 `postProcessBeanFactory` 方法，用于实现spring容器拓展的接口。

* `BeanPostProcessor`

  触发时机：AbstractAutowireCapableBeanFactory.initializeBean()，实例化之后初始化之前。

  ```java
  protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
  		if (System.getSecurityManager() != null) {
  			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
  				invokeAwareMethods(beanName, bean);
  				return null;
  			}, getAccessControlContext());
  		}
  		else {
        //BeanNameAware
        //BeanClassLoaderAware
        //BeanFactoryAware
  			invokeAwareMethods(beanName, bean);
  		}
  
  		Object wrappedBean = bean;
  		if (mbd == null || !mbd.isSynthetic()) {
        //触发BeanPostProcessors前置通知
  			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  		}
  
  		try {
        //触发InitializingBean（先）
        //触发init-method（后）
  			invokeInitMethods(beanName, wrappedBean, mbd);
  		}
  		catch (Throwable ex) {
  			throw new BeanCreationException();
  		}
  		if (mbd == null || !mbd.isSynthetic()) {
        //触发BeanPostProcessors后置通知
  			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  		}
  
  		return wrappedBean;
  	}
  ```

  作用于Bean初始化前后，定义了以下两个方法：

  * `Object postProcessBeforeInitialization（Object bean, String beanName）`
  * `Object postProcessAftrInitialization（Object bean, String beanName）`

  下面列出的都继承自BeanPostProcessor

* `InstantiationAwareBeanPostProcessor`

  作用于Bean实例化前后，定义了以下三个方法：

  * `Object postProcessBeforeInstantiation（Class<?> beanClass, String beanName）` 

    触发时机：createBean.resolveBeforeInstantiation()

    ```java
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    		Object bean = null;
    		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    			// Make sure bean class is actually resolved at this point.
    			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    				Class<?> targetType = determineTargetType(beanName, mbd);
    				if (targetType != null) {
              //触发Object postProcessBeforeInstantiation（Class<?> beanClass, String 									//beanName）
              //因为SmartInstantiationAwareBeanPostProcessor继承自InstantiationAwareBeanPostProcessor，所以如果存在他的实现也一样
    					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
    					if (bean != null) {
                //触发所有的Object postProcessAftrInitialization（Object bean, String 										//beanName）
    						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
    					}
    				}
    			}
    			mbd.beforeInstantiationResolved = (bean != null);
    		}
    		return bean;
    	}
    ```

  * `boolean postProcessAfterInstantiation（Object bean, String beanName）` 

    触发时机：populateBean

    ```java
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    	for (BeanPostProcessor bp : getBeanPostProcessors()) {
    		if (bp instanceof InstantiationAwareBeanPostProcessor) {
    			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          //返回false直接返回，不进行下面的依赖注入
    			if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
    				continueWithPropertyPopulation = false;
    				break;
    			}
    		}
    	}
    }
    if (!continueWithPropertyPopulation) {
    	return;
    }
    ```

    

  * `postProcessProperties` 

    触发时机：populateBean

    ```java
    if (hasInstAwareBpps) {
    	if (pvs == null) {
    		pvs = mbd.getPropertyValues();
    	}
    	for (BeanPostProcessor bp : getBeanPostProcessors()) {
    		if (bp instanceof InstantiationAwareBeanPostProcessor) {
    			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
    			PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    			if (pvsToUse == null) {
    				if (filteredPds == null) {
    					filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    				}
            //更改属性值
    				pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
    				if (pvsToUse == null) {
    					return;
    				}
    			}
    			pvs = pvsToUse;
    		}
    	}
    }
    ```

    

* `MergedBeanDefinitionPostProcessor `

  定义了以下两个方法：

  * `postProcessMergedBeanDefinition `

    触发时机：doCreateBean.applyMergedBeanDefinitionPostProcessors()

  * `resetBeanDefinition`

* `DestructionAwareBeanPostProcessor `

  Bean对象销毁的前置回调，定义了以下两个方法：

  * ` postProcessBeforeDestruction `

  * ` requiresDestruction `

    触发时机：doCreateBean.registerDisposableBeanIfNecessary.requiresDestruction()

* `SmartInstantiationAwareBeanPostProcessor`

  继承自`InstantiationAwareBeanPostProcessor`

  定义了以下方法：

  * `predictBeanType`

    触发时机：refresh.registerListeners.getBeanNamesForType.isTypeMatch.predictBeanType()

  * `determineCandidateConstructors`

    触发时机：createBeanInstance.determineConstructorsFromBeanPostProcessors()

    ```java
    protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
    			throws BeansException {
    
    		if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
    			for (BeanPostProcessor bp : getBeanPostProcessors()) {
    				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
    					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
    					Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
    					if (ctors != null) {
    						return ctors;
    					}
    				}
    			}
    		}
    		return null;
    	}
    ```

  * `getEarlyBeanReference`

    触发时机：doCreateBean.getEarlyBeanReference()

    ```java
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&isSingletonCurrentlyInCreation(beanName));
    	if (earlySingletonExposure) {
    		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    	}
    ```

# Aware接口

该接口用于获取容器的相关属性服务，常见的aware接口：

* BeanNameAware
* BeanFactoryAware
* ApplicationContextAware
* MessageSourceAware
* ApplicationEventPublisherAware
* ResourceLoaderAwareed

# Ordered 接口

```java
public interface Ordered {
    int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;
    int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
    int getOrder();
}

//OrderComparator类compare()
//若2个对象中有一个对象实现了PriorityOrdered接口，那么这个对象的优先级更高；
//若2个对象都是PriorityOrdered或Ordered接口的实现类，那么getOrder方法返回值越低，优先级越高
public int compare(Object o1, Object o2) {
    boolean p1 = (o1 instanceof PriorityOrdered);
    boolean p2 = (o2 instanceof PriorityOrdered);
    if (p1 && !p2) {
        return -1;
    }
    else if (p2 && !p1) {
        return 1;
    }
	int i1 = getOrder(o1);
    int i2 = getOrder(o2);
    return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;  
}
```

# Spring xml schema拓展

* 创建一个 XML Schema 文件，描述自定义的合法构建模块，也就是xsd文件。

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <xsd:schema xmlns="http://www.mycompany.com/schema/myns"
          xmlns:xsd="http://www.w3.org/2001/XMLSchema"
          xmlns:beans="http://www.springframework.org/schema/beans"
          targetNamespace="http://www.mycompany.com/schema/myns"
          elementFormDefault="qualified"
          attributeFormDefault="unqualified">
  
    <xsd:import namespace="http://www.springframework.org/schema/beans"/>
  
    //配置标签名
    <xsd:element name="dateformat">
    	<xsd:complexType>
      	<xsd:complexContent>
        	<xsd:extension base="beans:identifiedType">
            //配置标签属性
          	<xsd:attribute name="" type=""/>
          </xsd:extension>
        </xsd:complexContent>
      </xsd:complexType>
    </xsd:element>
  </xsd:schema>
  ```

* 自定义个处理器类，并实现`NamespaceHandler`接口。

  `NamespaceHandler`接口用于解析配置文件中遇到的特定命名空间的所有元素，该接口提供了如下三个方法：

  * `init()`: NamespaceHandler使用之前调用，完成NamespaceHandler的初始化。
  * `BeanDefinition parse(Element, ParserContext)`: 当遇到顶层元素时被调用。
  * `BeanDefinition decorate(Node,BeanDefinitionHandler,ParserContext)`: 当遇到一个属性或者嵌套元素的时候调用。

  Spring提供了该接口的默认实现类`NamespaceHandlerSupport`，所以我们呢只需要继承该接口，并在init方法中为元素注册解析器就行了。

  ```java
  public class xxxNamespaceHandler extends NamespaceHandlerSupport { 
  	public void init() { 
      registerBeanDefinitionParser("元素名", new xxxDefinitionParser(); 
    }
  }
  ```

* 自定义一个或多个解析器，实现`BeanDefinitionParser`接口。

  Spring提供了该接口的实现类`AbstractSingleBeanDefinitionParser`，通过继承该实现类，并实现以下两个方法便能实现自定义的解析器。

  * `Class getBeanClass(Element)`
  * `void doParse(Element element,BeanDefinitionBuilder builder)`

* 注册上面的组件到Spring IOC容器中。

  为了让Spring在解析xml的时候能够感知到我们的自定义元素，我们需要把`NamespaceHandler`和`xsd文件`放到2个指定的配置文件中，这2个文件都位于`META-INF`目录中：

  * `META-INF/spring.handlers`：包含XML Schema URI到命名空间处理程序类的映射。
  * `META-INF/spring.schemas`：包含XML Schema xsd到类路径资源的映射。

# Spring 自定义注解

**自定义注解是不需要在配置文件中显示配置的**

```java
@Target({ ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyAutoFillin {
	//对应注解中的各个属性
    String name();

    String method();
}
```

JDK自带以下几种元注解：

* @Target

  用于描述注解的使用范围，有一个枚举**ElementType**来指定,具体如下:

  1. CONSTRUCTOR:用于描述构造器
  2. FIELD:用于描述域
  3. LOCAL_VARIABLE:用于描述局部变量
  4. METHOD:用于描述方法
  5. PACKAGE:用于描述包
  6. PARAMETER:用于描述参数
  7. TYPE:用于描述类、接口(包括注解类型) 或enum声明

* @Retention

  描述需要在什么级别保存该注释的信息

  1.  RetentionPolicy.SOURCE：注解信息只保留在源文件中，编译成class文件注解信息丢失
  2.  RetentionPolicy.CLASS：注解信息保留到class文件阶段，jvm加载后注解信息丢失
  3.  RetentionPolicy.RUNTIME：注解信息保留到jvm加载后

* @Documented

  javadoc生成文档时，注解也会生成相应文档

* @Inherited

  如果在注释类型声明中存在 Inherited 元注释，并且用户在某一类声明中查询该注释类型，同时该类声明中没有此类型的注释，则将在该类的超类中自动查询该注释类型。 

# Spring 事务监听

* **@TransactionalEventListener**

  在相应的事件处理方法上添加该注解，并利用phase属性指定事务姐阶段，表示当事务处于当前阶段该监听方法才会执行，该属性提供了以下四种类型：

  * TransactionPhase.BEFORE_COMMIT
  * TransactionPhase.AFTER_COMMIT（默认）
  * TransactionPhase.AFTER_ROLLBACK
  * TransactionPhase.AFTER_COMPLETION

*  **TransactionSynchronizationManager** （上面注解的底层实现）

  ```java
  @EventListener
  void onSaveUserEvent(SaveUserEvent event) {
      TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
          @Override
          public void afterCommit() {
              //处理代码
          }
      });
  }
  ```

