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

