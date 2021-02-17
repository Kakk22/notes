# Spring源码深度解析

## 第5章 bean的加载

### 1、doGetBean方法

调用`ClassPathXmlApplicationContext`的`getBean`方法

核心实际调用的是`doGetBean`方法

![Snipaste_2021-01-26_09-32-35](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/Snipaste_2021-01-26_09-32-35.png)

```java
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {
	
   //提出对应的beanName 
   String beanName = transformedBeanName(name);
   Object bean;

   // 检查缓存中或者实例工厂中是否有对应的实例
   // 因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
   // Spring创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory早期曝光
   // 也就是将ObjectFactory加入缓存中，一旦下一个bean创建需要依赖上个bean则直接使用ObjectFactory
   // 直接尝试从缓存或者将SingletonFactories 中的ObjectFactory中获取
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
       // 返回对应的实例，有时候存在诸如BeanFactory的情况并不是直接返回实例本身
       // 而是返回指定方法返回的实例 
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // 只有在单例情况才会尝试解决循环依赖，原型模式情况下，如果存在
      // A中有B属性，B中有A属性，那么当依赖注入的时候，就会产生当A还未创建完成的时候因为
      // 对于B的创建再次返回创建A， 造成循环依赖，也就是下面的情况。
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      BeanFactory parentBeanFactory = getParentBeanFactory();
       // 如果beanDefinitionMap 中 也就是在所以已经加载的类中不包括beanName则尝试从
       // parentBeanFactory中获取
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // 递归到BeanFactory中寻找
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
	  // 如果不是仅仅做类型检查则是创建Bean 这里要进行记录
      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
         RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // 若存在依赖 则需要递归实例化依赖的bean
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
                // 缓存调用依赖
               registerDependentBean(dep, beanName);
               try {
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // 实例化依赖的bean后便可以实例化mbd本身
         // 单例模式的创建
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // 如果是多例 则创建bean
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
            if (!StringUtils.hasLength(scopeName)) {
               throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
            }
            Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
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
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // 检查需要的类型是否符合bean的实际类型
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) { 
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```



bean的加载过程大致分为几个步骤：

1. 装换对应的beanName
2. 尝试从缓存中加载单例
3. bean的实例化
4. 原型模式的依赖检查（只有单例才会尝试解决循环依赖）
5. 检测parentBeanFactory
6. 将存储XML配置文件的GernericBeanDefinition装换为RootBeanDefinition
7. 寻找依赖
8. **针对不同的scope进行bean的创建**
9. 类型转换

### 2、FactoryBean 的使用

用户可以通过实现`FactoryBean`接口定制实例化`bean`逻辑

```java
public interface FactoryBean<T> {
   /**
   * 返回由FactoryBean创建的bean实例，如果isSingleton()返回true
   * 则该实例会放到Spring容器中实例缓存池中
   */
   @Nullable
   T getObject() throws Exception;

   /**
   * 返回由FactoryBean创建的bean类型
   */
   @Nullable
   Class<?> getObjectType();


   default boolean isSingleton() {
      return true;
   }

}
```

### 3、缓存中获取单例bean

检查缓存中或者实例工厂中是否有对应的实例
 因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
Spring创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory早期曝光
也就是将ObjectFactory加入缓存中，一旦下一个bean创建需要依赖上个bean则直接使用ObjectFactory
直接尝试从缓存或者将SingletonFactories 中的ObjectFactory中获取

```java
	@Override
	@Nullable
	public Object getSingleton(String beanName) {
		return getSingleton(beanName, true);
	}
 
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 检查缓存中是否存在实例
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // 如果为空 则锁定全局变量并进行处理
			synchronized (this.singletonObjects) {
                // 如果此bean正在加载则不处理
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
                    // 当某些方法需要提前初始化的时候则会调用addSingletonFactory方法将对应的ObjectFactory初始化策略存储在singletonFactories
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                        // 调用预先设定的getObject方法
						singletonObject = singletonFactory.getObject();
                        // 记录在缓存中 earlySingletonObjects和singletonFactories 互斥
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

#### 三级缓存解释

`singletonObjects`：用于保存BeanName和创建bean实例之间的关系。 bean name ---> bean instance

`singletonFactories`：用于保存BeanName和创建bean的工厂之间的关系。 bean name ----> ObjectFactory

`earlySingletonObjects`：也是保存BeanName和创建bean实例之间的关系。与SingletonObjects的不同之处在于，当一个单例bean放到这里后，那么当bean

还在创建过程中，就可以通过getBean方法获取到了。起目的是用来检测循环依赖引用。

`registeredSingletons`：用来保存当前所有已注册的bean

### 4、从bean的实例中获取对象

在getBean方法中，getObjectForInstance是个高频的使用方法，无论是从缓存中获取bean还是根据不同的scope策略加载bean。总之，我们得到bean的实例后要做的第一步就是调用这个方法来检测一下正确性，其实就是用于检测当前bean是不是FactoryBean类型的bean，**如果是，那么需要调用该bean对应的FactoryBean实例中的getObject()作为返回值。**

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// 如果指定的name是工厂相关的（以&为前缀）且beanInstance又不是FactoryBean类型则不通过
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// 现在我们有一个bean实例，这个实例可能会是正常的bean或者是FactoryBean
		// 如果是FactoryBean 我们使用它创建实例，但是如果用户想到直接获取工厂实例而不是工厂的getObject方法对应的实例
		// 那么传入的name应该加前缀&
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}
		// 加载FactoryBean
		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
            // 尝试从缓存中加载bean
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 到这里已经明确知道beanInstance 一定是FacotyBean类型
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// containsBeanDefinition检测beanDefinitionMap中也就是在所有已经加载的类中检测是否定义beanName
			if (mbd == null && containsBeanDefinition(beanName)) {
                // 将存储XML配置文件的GernericBeanDefinition装换为RootBeanDefinition
                // 如果指定beanName是子bean的话 同时会合并父类的相关属性
				mbd = getMergedLocalBeanDefinition(beanName);
			}
            // 是否是用户定义的而不是应用程序本身定义的
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

### 5、获取单例

如果在缓存中获取不到bean 

则是从头开始加载bean

```java
	/**
	 * Return the (raw) singleton object registered under the given name,
	 * creating and registering a new one if none registered yet.
	 * @param beanName the name of the bean
	 * @param singletonFactory the ObjectFactory to lazily create the singleton
	 * with, if necessary
	 * @return the registered singleton object
	 */
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
        // 全局变量需要同步
		synchronized (this.singletonObjects) {
            // 首先检查对应的bean是否已经加载过，因为singleton模式其实就是复用以创建的bean
            // 所以这一步是必须的
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
                // 如果为空才可以进行singleton的bean的初始化
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
                // 创建bean前记录正在创建状态
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
                    // 初始化bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
                    // 移除创建状态
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
                    // 添加到缓存
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

上述代码中主要的内容是：

1. 检查缓存是否已经加载过
2. 若没有加载，记录beanName正在加载状态
3. 加载单例前记录加载状态
4. 通过调用参数传入的ObjectFactory的getObject方法实例化bean
5. 加载单例后的处理方法调用
6. 将结果记录至缓存并删除加载bean过程中所记录的各种辅助状态
7. 返回处理结果

ObjectFactory的核心部分就是调用**createBean**方法

### 6、准备创建bean



```java
	/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 * @see #doCreateBean
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 锁定calss 根据设置的class属性或者根据className来解析class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 验证及准备覆盖的方法
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 给BeanPostProcessors一个机会来返回代理来替代真正的实例
            // 如果前置处理器返回一个bean则直接返回该代理bean
            // 不再初始化原始bean
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
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
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

具体的功能步骤：

1. 根据设置的class属性或者根据className来解析Class
2. 对override属性进行标记及验证
3. **应用初始化前的后处理器**，解析指定bean是否存在初始化前的短路操作
4. 创建bean

### 7、循环依赖

```java
	//核心
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);//对象工厂 三级缓存
				this.earlySingletonObjects.remove(beanName);// 早期对象缓存 二级缓存
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

![image-20210130152425408](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210130152425408.png)

详细见 ../面试/spring循环依赖

### 8、创建bean

 核心`doCreateBean`方法

```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 根据指定bean使用对应的策略创建新的实例。如：工厂方法、构造函数自动注入、简单初始化
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// 是否需要提早曝光：单例&允许循环依赖&当前bean正在创建中。检测循环依赖
		// 
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            // 为了避免后期循环依赖，可以再bean初始化完成前将创建实例的ObjectFactory加入工厂
            // 对bean再一次依赖引用，主要应用SmartInstantiationAware BeanPostProcessor
            // 其中AOP就是在这里将advice动态织入bean中，若没有则直接返回
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            // 对bean进行属性填充,将各个属性注入,其中，可以能存在依赖其他bean的属性，则会递归初始依赖bean
			populateBean(beanName, mbd, instanceWrapper);
            // 调用初始化方法 比如 init-method
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
            // earlySingletonReference 只有在检测到有循环依赖的情况下才不为空
			if (earlySingletonReference != null) {
                // 如果exposedObject没有在初始化方法中被改变,也就是没有被增强
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
                        // 检测依赖
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
                    // 因为bean创建后其所依赖的bean一定是已经创建的
                    // actualDependentBeans.不为空则表示当前bean创建后其依赖的bean却没有全部创建完，就是说还存在循环依赖
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
            // 根据scope注册bean
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}

```



`doCreateBean`的整体思路：

1. 如果是单例则需要清除缓存
2. 实例化bean，将beanDefinition转换为BeanWrapper
3. MergedBeanDefinitionPostProcessor的应用
4. 依赖处理
5. 属性填充
6. 循环依赖检查
7. 注册DisposableBean
8. 完成创建并返回

#### 8.1 创建bean的实例

核心方法`createBeanInstance` 

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		//解析class
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		
        // 如果工厂方法不为空，则使用工厂方法初始化策略
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
            // 一个类有多个构造函数，每个构造函数都有不同的参数，所以调用前需要先根据参数锁定构造函数或对应的工厂方法
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
        // 如果已经解析过则使用解析好的构造函数方法不需要再次锁定
		if (resolved) {
			if (autowireNecessary) {
                // 构造函数自动注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
                // 使用默认构造函数构造
				return instantiateBean(beanName, mbd);
			}
		}

		// 需要根据参数解析构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
            // 构造函数自动注入
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 使用默认构造函数构造
		return instantiateBean(beanName, mbd);
	}
```

主要逻辑：

1. 如果工厂方法不为空，则使用工厂方法初始化策略
2. 解析构造函数并进行构造函数实例化。一个类有多个构造函数，每个构造函数都有不同的参数，所以调用前需要先根据参数锁定构造函数或对应的工厂方法



##### 1.autowireConstructor

```java
	public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Constructor<?> constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;

		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached constructor...
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			if (argsToResolve != null) {
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
			}
		}

		if (constructorToUse == null || argsToUse == null) {
			// Take specified constructors, if any.
			Constructor<?>[] candidates = chosenCtors;
			if (candidates == null) {
				Class<?> beanClass = mbd.getBeanClass();
				try {
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
							"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
			}

			if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
				Constructor<?> uniqueCandidate = candidates[0];
				if (uniqueCandidate.getParameterCount() == 0) {
					synchronized (mbd.constructorArgumentLock) {
						mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
						mbd.constructorArgumentsResolved = true;
						mbd.resolvedConstructorArguments = EMPTY_ARGS;
					}
					bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
					return bw;
				}
			}

			// Need to resolve the constructor.
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;

			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor<?>> ambiguousConstructors = null;
			LinkedList<UnsatisfiedDependencyException> causes = null;

			for (Constructor<?> candidate : candidates) {
				int parameterCount = candidate.getParameterCount();

				if (constructorToUse != null && argsToUse != null && argsToUse.length > parameterCount) {
					// Already found greedy constructor that can be satisfied ->
					// do not look any further, there are only less greedy constructors left.
					break;
				}
				if (parameterCount < minNrOfArgs) {
					continue;
				}

				ArgumentsHolder argsHolder;
				Class<?>[] paramTypes = candidate.getParameterTypes();
				if (resolvedValues != null) {
					try {
						String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, parameterCount);
						if (paramNames == null) {
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
						}
						argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
								getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (logger.isTraceEnabled()) {
							logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next constructor.
						if (causes == null) {
							causes = new LinkedList<>();
						}
						causes.add(ex);
						continue;
					}
				}
				else {
					// Explicit arguments given -> arguments length must match exactly.
					if (parameterCount != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}

				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this constructor if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}

			if (constructorToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Could not resolve matching constructor " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
			}
			else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous constructor matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousConstructors);
			}

			if (explicitArgs == null && argsHolderToUse != null) {
				argsHolderToUse.storeCache(mbd, constructorToUse);
			}
		}

		Assert.state(argsToUse != null, "Unresolved constructor arguments");
		bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
		return bw;
	}
```



##### 2.instantiateBean

无参的构造函数实例化

```java
protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      if (System.getSecurityManager() != null) {
         beanInstance = AccessController.doPrivileged(
               (PrivilegedAction<Object>) () -> getInstantiationStrategy().instantiate(mbd, beanName, this),
               getAccessControlContext());
      }
      else {
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
      }
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}
```

直接调用实例化策略进行实例化即可

##### 3.实例化策略

`SimpleInstantiationStrategy`这个类的`instantiate`

```java
	@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// 如果有需要覆盖或者动态替换的方法则当然需要使用cglib进行动态代理
        // 因为可以再创建代理的同时将动态方法织入类中
        // 但是如果没有需要动态改变的方法，为了方便直接反射就可以了
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							constructorToUse = clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}

```

这里主要是如果设置了代理信息

则使用动态代理方法将拦截增强器设置进去

如果没有 则直接使用**反射创建**。

#### 8.2 记录创建bean的ObjectFactory

在`doCreateBean`这个方法中有一段代码

```java
		// 为避免后期循环依赖 可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) { 
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            // 对bean再一次依赖引用，主要应用SmartInstantiationAware BeanPostProcessor
            // 其中我们熟知的AOP就是在这里将advice动态织入bean中
            // 若没有则直接返回bean 不作任何处理
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

	// 这里作后置处理器的调用
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}

```



**Spring解决循环依赖的核心：**在B创建依赖A时通过ObjectFactory提供实例化方法来中断A中的属性填充，使B中持有的A仅仅是刚刚初始化并没有填充任何属性的A，而这正初始化A的步骤还是在最开始创建A的时候进行的，但是因为A与B中的A所代表的属性是一样的，所以在A中创建好的属性填充自然可以通过B中的A获取，这样就解决了循环依赖的问题。

#### 8.3 属性注入

在`doCreateBean`方法中初始化完bean后开始填充bean的属性

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // 没有可填充的属性
         return;
      }
   }

   // 给InstantiationAwareBeanPostProcessors最后一次机会在属性设置前改变bean
   // 如：可以用来支持属性注入的类型
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               return;
            }
         }
      }
   }

   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

   int resolvedAutowireMode = mbd.getResolvedAutowireMode();
   if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // 根据名称自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // 根据类型自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }
   // 后置处理器已经初始化
   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   // 需要依赖检查
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

   PropertyDescriptor[] filteredPds = null;
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
                // 对所有需要依赖检查的属性进行后处理
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }

   if (pvs != null) {
       // 将属性应用到bean中
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

`populateBean`函数中提供了这样的处理流程：

1. InstantiationAwareBeanPostProcessors 处理器的postProcessAfterInstantiation函数的应用，此函数可以控制程序是否继续进行属性填充
2. 根据注入类型（byName/byType），提取依赖的bean，并统一存入PropertyValues中
3. 应用InstantiationAwareBeanPostProcessors处理器的postProcessPropertyValues的方法，对属性获取完毕填充前对属性的再次处理，典型应用是RequiredAnnotationBeanPostProcessor类中对属性的验证
4. 将所有PropertyValues中的属性填充至BeanWrapper中



##### 1.autowireByName

​	主要逻辑就是在传入的参数`pvs`中找到已经加载的bean，并递归初始化加入到`pvs`中

```java
	protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
		// 寻找bw中需要依赖注入的属性
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			if (containsBean(propertyName)) {
                // 递归初始化相关的bean
				Object bean = getBean(propertyName);
                // 添加属性
				pvs.add(propertyName, bean);
                // 注册依赖
				registerDependentBean(propertyName, beanName);
				if (logger.isTraceEnabled()) {
					logger.trace("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
```

##### 2.autowireByType

```java
	protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
        // 寻找bw中需要依赖注入的属性
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				// Don't try autowiring by type for type Object: never makes sense,
				// even if it technically is a unsatisfied, non-simple property.
				if (Object.class != pd.getPropertyType()) {
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// 探测指定属性的set方法
					boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                    // 解析指定beanName的属性所匹配的值，并把解析到的属性名称存储在autowiredBeanNames中，当属性存在多个封装bean时如：
                    // @autowired private List<A> aList； 将会找到所有匹配A类型的bean并将其注入
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
                        // 注册依赖
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isTraceEnabled()) {
							logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

##### 3.applyPropertyValues

程序允许到这里，已经完成了对所有注入属性的获取，但是获取的属性是以PropertyValues形式存在的。

还没有实例化到bean中，这一工作在`applyPropertyValues`中。

```java
	/**
	 * Apply the given property values, resolving any runtime references
	 * to other beans in this bean factory. Must use deep copy, so we
	 * don't permanently modify this property.
	 * @param beanName the bean name passed for better exception information
	 * @param mbd the merged bean definition
	 * @param bw the BeanWrapper wrapping the target object
	 * @param pvs the new property values
	 */
	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
            // 如果mpvs中的值已经被转化为对应的类型那么可以直接设置到beanwapper中
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
            // 如果不是按照原始的类型进行填充
			original = Arrays.asList(pvs.getPropertyValues());
		}

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				if (originalValue == AutowiredPropertyMarker.INSTANCE) {
					Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
					if (writeMethod == null) {
						throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
					}
					originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
				}
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
```

#### 8.4 初始化bean

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          // 激活aware
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       // 对特殊bean的处理:Aware BeanClassLoaderAware BeanFactoryAware
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       // 前置处理器应用
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       // 激活用户自定义的init方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       // 后置处理器应用
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

1. 激活aware方法

   Spring框架中提供了许多实现了Aware接口的类，这些类主要是为了辅助Spring访问容器中的数据，比如`BeanNameAware`，这个类能够在Spring容器加载的过程中将Bean的名字（id）赋值给变量。

   例如：**ApplicationContextAware**

   ApplicationContext可以获取容器中的bean，但是必须注入才能使用，当一些类不能注入的时候怎么才能获得bean呢？比如Utils中的类，通常不能直接通过注入直接使用ApplicationContext，此时就需要借助`ApplicationContextAware`这个接口了。

2. 处理器的应用

   **BeanPostProcessor**为bean的处理器。可以根据此在bean的初始化前或初始化后进行业务的处理。

3. 激活用户自定义init方法

   ```java
   	protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
   			throws Throwable {
   		// 首先检查是否是InitializingBean 如果是的话要调用afterPropertiesSet方法
   		boolean isInitializingBean = (bean instanceof InitializingBean);
   		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
   			if (logger.isTraceEnabled()) {
   				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
   			}
   			if (System.getSecurityManager() != null) {
   				try {
   					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
   						((InitializingBean) bean).afterPropertiesSet();
   						return null;
   					}, getAccessControlContext());
   				}
   				catch (PrivilegedActionException pae) {
   					throw pae.getException();
   				}
   			}
   			else {
                   // 属性初始化后的处理
   				((InitializingBean) bean).afterPropertiesSet();
   			}
   		}
   
   		if (mbd != null && bean.getClass() != NullBean.class) {
   			String initMethodName = mbd.getInitMethodName();
   			if (StringUtils.hasLength(initMethodName) &&
   					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
   					!mbd.isExternallyManagedInitMethod(initMethodName)) {
                   // 调用自定义初始化的方法
   				invokeCustomInitMethod(beanName, bean, mbd);
   			}
   		}
   	}
   ```

#### 8.5 注册DisposableBean

销毁bean除了配置属性`destory-method`方法外

用户还可以注册后处理器`DestructionAwareBeanPostProcessors`

```java
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
				registerDisposableBean(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// A bean with a custom scope...
				Scope scope = this.scopes.get(mbd.getScope());
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
				}
				scope.registerDestructionCallback(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
		}
	}
```

### 9.bean的生命周期图

![image-20210202104014894](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210202104014894.png)

## 第6章 容器的功能扩展

### 1、扩展功能

使用`ApplicationContext`加载XML

```java
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
										|
                                        |
                                      
      public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
            //核心方法 refresh
			refresh();
		}
	}

```

`refresh`函数包含了几乎`ApplicationContext`中提供的全部功能

`AbstractApplicationContext`中的`refresh`实现了所有逻辑

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备刷新上下文环境
			prepareRefresh();

			// 初始化BeanFactory，并进行XML文件的读取
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 对BeanFactory进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// 子类覆盖方法进行处理
				postProcessBeanFactory(beanFactory);

				// 激活各种BeanFactory处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册拦截Bean创建的Bean处理器，这是只是注册，真正的调用在getBean时候
				registerBeanPostProcessors(beanFactory);

				// 为上下文初始化Message源，即不同语言的消息体，国际化处理
				initMessageSource();

				// 初始化应用消息广播器，并放入“applicationEventMulticaster”bean中
				initApplicationEventMulticaster();

				// 留给子类来初始化其他Bean
				onRefresh();

				// 在所有注册的bean中查找Listener bean 注册到消息广播器中
				registerListeners();

				// 实例化剩下的单实例（非惰性的）
				finishBeanFactoryInitialization(beanFactory);

				// 完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程
                // 同时发出ContextRefreshEvent 通知别人
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```

**ClassPathXmlApplicationContext** 初始化的步骤：

1. 初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证

2. 初始化BeanFactory,并对XML文件读取

   这一步将会复用BeanFactory中的配置文件读取解析及其他功能，这一步后ClassPathXmlApplicationContext已经包含了BeanFactory所提供的功能，也就可以进行bean的提取等基础操作。

3. 对BeanFactory进行各种功能填充

   **@Qualifer与@Autowired**在这一步骤中增加的支持。

4. 子类覆盖方法做额外的处理

   这里留了一个空方法。留给扩展使用。 空方法`postProcessBeanFactory`

5. 激活各种BeanFactory处理器

6. 注册拦截Bean创建的Bean处理器，这是只是注册，真正的调用在getBean时候

7. 为上下文初始化Message源，即不同语言的消息体，国际化处理

8. 初始化应用消息广播器，并放入“applicationEventMulticaster”bean中

9. 留给子类来初始化其他Bean

10. 在所有注册的bean中查找Listener bean 注册到消息广播器中

11. 实例化剩下的单实例（非惰性的）

12. 完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程。同时发出ContextRefreshEvent 通知别人

### 2、加载BeanFactory

![image-20210202133903946](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210202133903946.png)

上面步骤分别为：

1. 创建DefaultListableBeanFactory
2. 指定序列化iD
3. 定制BeanFactory
4. 加载BeanDefinition
5. 使用全局变量记录BeanFactory实例

### 3、功能扩展

进入函数PrepareBeanFactory前，spring已经完成了对配置的解析，而ApplicationContext在功能上的扩展也由此展开

```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置当前的类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
        // 设置beanFactory的表达式语言处理器，Spring3增加了表达式语言的支持
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        // 为beanFactory增加一个默认的propertyEditor 这个主要是对bean的属性等设置管理的一个工具
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 添加BeanPostProcessor
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        
        //设置几个忽略自动装配的接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// 配置几个自动装配的特殊规则
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);
        
        // 注册早期的后处理器以将内部bean检测为ApplicationListeners。
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));


		// 增加对AspectJ的支持
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 添加默认的系统环境bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

上面函数主要进行了几方面的扩展：

1. 增加对SpEL语言的支持
2. 增加对属性编辑器的支持
3. 增加对一些内置类，比如EnvironmentAware 的信息注入
4. 设置了依赖功能可忽略的接口
5. 注册一些固定依赖的属性
6. 增加AspectJ的支持
7. 将相关的环境变量及属性注册以单例模式注册

### 4、 BeanFactory的后处理

#### 4.1激活注册的BeanFactoryPostProcessor

`BeanFactoryPostProcessor`和`BeanPostProcessor`类似，可以对`bean`的定义（配置元数据）进行处理。就是说，Spring ioc容器允许`BeanFactoryPostProcessor`**在容器实例化任何其他`bean`之前读取配置元数据**，并有可能修改它。

`invokeBeanFactoryPostProcessors`**核心方法**

```java
public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any.
   Set<String> processedBeans = new HashSet<>();
	// 对BeanDefinitionRegistry类型处理
   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
	
       /**
       * 硬编码注册后处理器
       */
      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            regularPostProcessors.add(postProcessor);
         }
      }

      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) {
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory);
         registryProcessors.addAll(currentRegistryProcessors);
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let the bean factory post-processors apply to them!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   beanFactory.clearMetadataCache();
}
```

### 5、注册BeanPostProcessor

这里仅仅实现的是注册，而不是调用。真正的调用在bean的实例化阶段进行的。

`registerBeanPostProcessors`核心方法，主要逻辑是注册实现`BeanPostProcessors`的类。并且在注册是要根据**有无顺序**进行排序

```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// BeanPostProcessorChecker 是一个普通的信息打印，可能会有些情况
		// 当spring的配置中的后处理还没有被注册就已经开始了bean的初始化时
		// 便会打印出BeanPostProcessorChecker中设定的信息
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		//PriorityOrdered 保证顺序
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
        // 使用order保证顺序
		List<String> orderedPostProcessorNames = new ArrayList<>();
        // 无序
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// 第一步，注册所有实现了PriorityOrdered的BeanPostProcessor
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// 第二步，注册所有实现Ordered的BeanPostProcessor
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// 第三步，注册所有无序的BeanPostProcessor
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
		  		internalPostProcessors.add(pp);
			}
		}
stProcessors(beanFactory, internalPostProcessors);

		// 添加 ApplicationListeners 探测器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

### 6、初始化ApplicationEventMulticaster

#### Spring事件监听的简单用法

1. 定义监听事件

   ```java
   /**
    * @author 陈一锋
    * @date 2021/1/22 10:10
    **/
   public class TestEvent extends ApplicationEvent {
   
       /**
        * Create a new {@code ApplicationEvent}.
        *
        * @param source the object on which the event initially occurred or with
        *               which the event is associated (never {@code null})
        */
       public TestEvent(Object source) {
           super(source);
       }
   }
   ```

2. 定义监听器

   ```java
   /**
    * @author 陈一锋
    * @date 2021/1/22 10:49
    **/
   @Component
   public class TestListener implements ApplicationListener<TestEvent> {
   
       /**
        * Handle an application event.
        *
        * @param event the event to respond to
        */
       @Override
       public void onApplicationEvent(TestEvent event) {
           // 接收到事件
           System.out.println("接收到事件通知:"+event.getClass().getName());
           System.out.println(event.getSource());
           System.out.println(event.getMsg());
       }
   }
   
   ```

3. 测试

   ```java
   /**
    * @author 陈一锋
    * @date 2021/1/22 10:51
    **/
   public class Client {
   
       public static void main(String[] args) {
           // 1. 初始化容器
           ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
           // 2. 创建自定义事件
           TestEvent testEvent = new TestEvent("myEvent","messages");
           //发布事件
           context.publishEvent(testEvent);
       }
   }
   
   ```

 主要步骤就是定义好事件及监听器，事件触发后的处理流程

然后使用容器`publishEvent`方法发布事件

#### **源码**

`refresh`中初始化广播器`initApplicationEventMulticaster();`


```java
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            // 如果自定义了事件广播器,则使用用户自定义的事件广播器
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
            // 如果没有自定义事件广播器，则默认使用SimpleApplicationEventMulticaster
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

当发布事件时

```java
	protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
            //这一段是核心，获取事件广播器并发布事件
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}

```

```java
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        // 如果有线程池则使用线程池
		Executor executor = getTaskExecutor();
        // 获取所有符合的监听器并遍历发布事件
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

### 7、初始化非延迟加载单例

初始化剩下的非延迟bean

`refresh`中的`finishBeanFactoryInitialization(beanFactory)`方法

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 初始化上下文中的ConversionService 该类主要是转换器
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// 冻结所有bean的定义，说明注册的bean将不被修改或任何进一步的处理
		beanFactory.freezeConfiguration();

		// 初始化剩下的非延迟加载的单实例
		beanFactory.preInstantiateSingletons();
	}
```

初始化非延迟加载

`ApplicationContext`默认为启动时将所有的`bean`提前进行实例化。

好处是配置有任何错误就立刻被发现

```java
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// 遍历所有注册的bean并实例化
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
                    //核心方法
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

### 8、finishRefresh

主要逻辑是

1. 初始化所有实现Lifecycle接口的bean
2. 调用其start方法开启生命周期
3. 发布容器刷新事件

```java
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

### spring bean的生命周期图

![](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20210215222109666.png)

## 第7章 AOP

### 1、动态AOP使用示例

创建用于拦截的bean

```java
public class HelloServiceImpl implements HelloService {

    @Override
    public void hello(){
        System.out.println("hello");
    }
}
```

创建Advisor

```java
@Aspect
public class AspectDemo {

    /**
     * 所有类的hello方法会被执行aop
     */
    @Pointcut("execution(* *.hello(..))")
    public void pointcut() {
    }

    @Before("pointcut()")
    public void before() {
        System.out.println("before");
    }

    @After("pointcut()")
    public void after() {
        System.out.println("after");
    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) {
        System.out.println("before around");
        Object result = null;
        try {
            result = proceedingJoinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        System.out.println("after around");
        return result;
    }
}
```

引入依赖

```xml
        <!--aop-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>5.2.8.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjrt</artifactId>
            <version>1.8.9</version>
        </dependency>

        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.8.9</version>
        </dependency>
```

修改配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd"
       default-autowire="byName">

    <aop:aspectj-autoproxy/>


    <bean id="helloService" class="com.cyf.aop.HelloServiceImpl"/>
    <bean id="aspect" class="com.cyf.aop.AspectDemo"></bean>
</beans>
```

结果输出

```
before around
before
hello
after
after around
```

结果Spring对所有类的hello方法进行增强

### 2、动态AOP自定义标签

