# spring如何解决循环依赖

1、如果是构造器注入，无法解决循环依赖问题

2、使用setter注入

setter注入：Spring是先将Bean对象实例化【依赖无参构造函数】--->再设置对象属性

![image-20200530233215921](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200530233215921.png)

Spring先用构造器实例化Bean对象----->将实例化结束的对象放到一个Map中，并且Spring提供获取这个未设置属性的实例化对象的引用方法。结合我们的实例来看，，当Spring实例化了StudentA、StudentB、StudentC后，紧接着会去设置对象的属性，此时StudentA依赖StudentB，就会去Map中取出存在里面的单例StudentB对象，以此类推，不会出来循环的问题



先去缓存里找Bean，没有则实例化当前的Bean放到Map，如果有需要依赖当前Bean的，就能从Map取到。

```java
    /**
     * 放置创建好的bean Map
     */
    private static Map<String, Object> cacheMap = new HashMap<>(2);

    public static void main(String[] args) {
        // 假装扫描出来的对象
        Class[] classes = {A.class, B.class};
        // 假装项目初始化实例化所有bean
        for (Class aClass : classes) {
            getBean(aClass);
        }
        // check
        System.out.println(getBean(B.class).getA() == getBean(A.class));
        System.out.println(getBean(A.class).getB() == getBean(B.class));
    }

    @SneakyThrows
    private static <T> T getBean(Class<T> beanClass) {
        // 本文用类名小写 简单代替bean的命名规则
        String beanName = beanClass.getSimpleName().toLowerCase();
        // 如果已经是一个bean，则直接返回
        if (cacheMap.containsKey(beanName)) {
            return (T) cacheMap.get(beanName);
        }
        // 将对象本身实例化
        Object object = beanClass.getDeclaredConstructor().newInstance();
        // 放入缓存
        cacheMap.put(beanName, object);
        // 把所有字段当成需要注入的bean，创建并注入到当前bean中
        Field[] fields = object.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            // 获取需要注入字段的class
            Class<?> fieldClass = field.getType();
            String fieldBeanName = fieldClass.getSimpleName().toLowerCase();
            // 如果需要注入的bean，已经在缓存Map中，那么把缓存Map中的值注入到该field即可
            // 如果缓存没有 继续创建
            field.set(object, cacheMap.containsKey(fieldBeanName)
                    ? cacheMap.get(fieldBeanName) : getBean(fieldClass));
        }
        // 属性填充完成，返回
        return (T) object;
    }
```



# Spring事务传播，事物隔离级别

 **事务属性的种类：**  **传播行为、隔离级别、只读和事务超时**

 

**a)**  **传播行为定义了被调用方法的事务边界。**

 

| **传播行为**                  | **意义**                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| **PROPERGATION_MANDATORY**    | **表示方法必须运行在一个事务中，如果当前事务不存在，就抛出异常** |
| **PROPAGATION_NESTED**        | **表示如果当前事务存在，则方法应该运行在一个嵌套事务中。否则，它看起来和 PROPAGATION_REQUIRED** **看起来没什么俩样** |
| **PROPAGATION_NEVER**         | **表示方法不能运行在一个事务中，否则抛出异常**               |
| **PROPAGATION_NOT_SUPPORTED** | **表示方法不能运行在一个事务中，如果当前存在一个事务，则该方法将被挂起** |
| **PROPAGATION_REQUIRED**      | **表示当前方法必须运行在一个事务中，如果当前存在一个事务，那么该方法运行在这个事务中，否则，将创建一个新的事务** |
| **PROPAGATION_REQUIRES_NEW**  | **表示当前方法必须运行在自己的事务中，如果当前存在一个事务，那么这个事务将在该方法运行期间被挂起** |
| **PROPAGATION_SUPPORTS**      | **表示当前方法不需要运行在一个是事务中，但如果有一个事务已经存在，该方法也可以运行在这个事务中** |

**b)**  **隔离级别**

**在操作数据时可能带来 3** **个副作用，分别是脏读、不可重复读、幻读。为了避免这 3** **中副作用的发生，在标准的 SQL** **语句中定义了 4** **种隔离级别，分别是未提交读、已提交读、可重复读、可序列化。而在 spring** **事务中提供了 5** **种隔离级别来对应在 SQL** **中定义的 4** **种隔离级别，如下：**

| **隔离级别**                   | **意义**                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| **ISOLATION_DEFAULT**          | **使用后端数据库默认的隔离级别**                             |
| **ISOLATION_READ_UNCOMMITTED** | **允许读取未提交的数据（对应未提交读），可能导致脏读、不可重复读、幻读** |
| **ISOLATION_READ_COMMITTED**   | **允许在一个事务中读取另一个已经提交的事务中的数据（对应已提交读）。可以避免脏读，但是无法避免不可重复读和幻读** |
| **ISOLATION_REPEATABLE_READ**  | **一个事务不可能更新由另一个事务修改但尚未提交（回滚）的数据（对应可重复读）。可以避免脏读和不可重复读，但无法避免幻读** |
| **ISOLATION_SERIALIZABLE**     | **这种隔离级别是所有的事务都在一个执行队列中，依次顺序执行，而不是并行（对应可序列化）。可以避免脏读、不可重复读、幻读。但是这种隔离级别效率很低，因此，除非必须，否则不建议使用。** |



**c)**  **只读**

**如果在一个事务中所有关于数据库的操作都是只读的，也就是说，这些操作只读取数据库中的数据，而并不更新数据，那么应将事务设为只读模式（ READ_ONLY_MARKER** **） ,** **这样更有利于数据库进行优化** **。**

**因为只读的优化措施是事务启动后由数据库实施的，因此，只有将那些具有可能启动新事务的传播行为 (PROPAGATION_NESTED** **、 PROPAGATION_REQUIRED** **、 PROPAGATION_REQUIRED_NEW)** **的方法的事务标记成只读才有意义。**

**如果使用 Hibernate** **作为持久化机制，那么将事务标记为只读后，会将 Hibernate** **的 flush** **模式设置为 FULSH_NEVER,** **以告诉 Hibernate** **避免和数据库之间进行不必要的同步，并将所有更新延迟到事务结束。**

**d)**  **事务超时**

**如果一个事务长时间运行，这时为了尽量避免浪费系统资源，应为这个事务设置一个有效时间，使其等待数秒后自动回滚。与设**

**置“只读”属性一样，事务有效属性也需要给那些具有可能启动新事物的传播行为的方法的事务标记成只读才有意义。**