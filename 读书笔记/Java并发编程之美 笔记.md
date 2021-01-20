# Java并发编程之美 笔记

## 第一章

### 1.1线程和进程

**进程**是系统进行资源分配的和调度的基本单位。**线程**是进程的一个执行路径。

进程中多个线程共享进程的资源。

**线程**是**CPU分配**的基本单位。

![image-20200710193944051](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200710193944051.png)

 多个线程共享堆和方法区资源。每个线程拥有自己的程序计数器和栈。

`程序计数器：是一块内存区域，用来记录线程当前要执行的指令地址`

`栈：存储线程的局部变量，是私有的 别的线程访问不了`

`堆：进程最大的内存，是共享的。主要存放new的新对象	`

`方法区：存放JVM加载的类，常量和静态变量等信息，是线程共享的`

### 1.2线程的创建与运行

1、继承Thread类

重写run方法，调用start()方法启动新线程。

**调用start()方法后处于就绪状态 等待获取cpu资源，获取cpu资源后才处于运行状态**

2、实现Runnable接口

重写run()方法。 

3、继承Callable<T>接口

**重写call()方法，可以有返回值**

```java
public static class CallTask implements Callable<String>{
    
    @Orriver
    public String call() throw Exception{
        retrun "hello";
    }
}

public class Test{
    public static void main(String args[]){
        FutureTask<String> futureTask = new FutureTask(new CallTask());
        new Thread(futureTask).start();
        try{
            String s = futureTask.get();
            Systrm.out.println(s);
        }catch(ExceptionException e){
            e.printStackTrace();
        }
    }
}
```

### 1.3线程通知和等待

当线程调用共享变量的wait（）方法时，线程被堵塞挂起，直到发生

1. 其他线程调用共享变量的notify()或者notfiyAll()方法
2. 其他线程调用该线程的interrupt()方法，并抛出interruptedException异常。

```java
//线程可以不经过唤醒，或者中断或者超时，从挂起状态变成变成运行状态，这种情况比较少见，称为虚假唤醒，解决的方法就是不断循环不满足条件则继续等待
synchronized(obj){
    while(条件不足){
        obj.wait();
    }
}
```

**线程调用共享变量的wait()方法后只会释放当前共享变量上的锁，如果线程还拥有其他共享变量的锁，则不会释放。**

### 1.4等待线程执行终止的join方法

join是Thread类无参无返回值的方法

调用此方法后会**等待线程执行完毕后返回继续执行**。

```java
package com.cyf.study.concurrent.demo;

/**
 * @author by cyf
 * @date 2020/7/11.
 */
public class JoinDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread threadOne = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("threadOne end");
        });

        Thread threadTwo = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("threadTwo end");
        });

        threadOne.start();

        threadTwo.start();
        System.out.println("wait all child thread over");

        threadOne.join();
        threadTwo.join();
        System.out.println("all child thread over");
    }
}
```

当线程被调用join方法堵塞时，如果其他线程调用Interrupt()方法时 会抛出interruptException异常。

### 1.5 sleep方法

Thread静态方法sleep()。线程调用此方法时会睡眠，不参与CPU调度，不会释放锁。

睡眠时间到后重新参与CPU调度。

### 1.6 让出CPU执行权的yield方法

比较少用。调用此方法时让出当前线程剩余的时间段。

与sleep的区别：

当线程sleep时，线程被堵塞挂起，在这段时间内线程调度器不会去调用该线程。

调用yield方法时，线程让出自己剩余的时间段，并没有堵塞挂起，处于就绪状态，线程调度器下一次可能调度到当前线程。

### 1.7线程中断

![image-20200711171856701](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200711171856701.png)

### 1.8线程上下文切换

### 1.9 死锁

死锁是两个或两个以上的线程，因为争夺资源互相等待的现象，在无外力作用下，这些线程会一直等待。

死锁产生的条件：

1. **互斥条件**：指线程对己经获取到的资源进行排它性使用，即该资源同时只由 一个线

   程占用

2. **请求并持有**：指一个 线程己经持有了至少一 个资源 但又提出了新的资源请求

   而新资源己被其他线程占有，所以当前线程会被阻塞，但阻塞的同时并不释放自

   己经获取的资源。

3. **不可剥夺条件**：指线程获取到的资源在自己使用完之前不能被其 线程抢占 只有

   在自己使用完 后才 释放该资源。

4. **环路等待条件**：指在发生死锁时 必然存在一个线程→资源的环形链 即线程集合

   {TO , TL T2 ，…， Tn ｝中 TO 正在等待一 Tl 占用 资源 Tl 正在等待 T2

   用的资源，……Tn 在等 己被 TO 资源。

如何避免死锁：

破坏其中一个条件即可：**目前只有请求并持有和环路等待条件是可以被破坏**

![image-20200711173344047](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200711173344047.png)

总结：资源分配要有序

### 1.10 守护线程

### 1.11 ThreadLocal

创建一下ThreadLocal 变量时，每一个线程访问此变量都会有这个变量的一个本地副本。

![image-20200712191440357](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200712191440357.png)

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

```

thread类里面有两个ThreadLocalMap 它其实就是特殊的HashMap 存放ThreadLocal绑定的值 可能一个线程有多个ThreadLocal值

**总结：如图 在每个线程 都有 threadLocals 的成员变量 该变量类型为 HashMap key 为我们定义 ThreadLocal 变量 this，value 则为我们使用 set 方法设置 每个线程 变量存放在线程自己的内存变量 threadLocals 中，如果当前线程一直不消亡 那么这些本地变量会一直存在 所以可能会造成内存溢出。使用完毕后要记得 ThreadLocal remove 方法 除对应线程 threadLocals 中的变量。**

## 第二章 并发编程基础

### 2.4 java共享变量的内存可见性问题

java内存模型

![image-20200712201836467](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200712201836467.png)

![image-20200712201932558](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200712201932558.png)

### 2.6 volatile关键字

**被volatile关键字修饰的变量，线程在修改此变量时，会把修改值直接刷新到主内存中，保证了多线程共享变量的可见性。但并不保证原子性。**

![image-20200712202205404](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200712202205404.png)

### 2.8 CAS操作

CAS 即 compare and swap  比较并且交换



![image-20200727220253972](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727220253972.png)



### 2.9 Unsafe类

Unsafe 类是JDK的rt.jar 提供了硬件级别的原子性操作。

Unsafe 类 方法都是native方法

![image-20200727220646756](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727220646756.png)

```java
package com.cyf.study.concurrent.demo;

import sun.misc.Unsafe;

import java.lang.reflect.Field;

/**
 * @author by cyf
 * @date 2020/7/27.
 */
public class TestUnsafe {
    static final Unsafe unsafe;
    static final long stateOffset;

    private volatile long state = 0;
    static {
        try {
            //利用反射获取Unsafe实例，Class （反射的入口）、Method （成员方法）、Field （成员变量），
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            unsafe = (Unsafe) field.get(null);

            stateOffset = unsafe.objectFieldOffset(TestUnsafe.class.getDeclaredField("state"));
        }catch (Exception e){
            System.out.println(e.getLocalizedMessage());
            throw new Error(e);
        }
    }

    public static void main(String[] args) {
        TestUnsafe testUnsafe = new TestUnsafe();
        boolean b = unsafe.compareAndSwapInt(testUnsafe, stateOffset, 0, 1);
        System.out.println(b);
    }
}
```

### 2.10 java指令重排序

![image-20200727222520579](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727222520579.png)

![image-20200727223002732](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727223002732.png)

### 2.12 锁

2.12.1 悲观锁和乐观锁

悲观锁：修改数据时总是觉得别人也会来修改数据，直接加排他锁 锁定

乐观锁： 加一个版本号 每次跟新对比版本号相同不相同，如果相同则修改版本号加一，不相同则重新尝试

2.12.2 公平锁和非公平锁

根据线程获取锁的抢占机制，锁可以分为公平锁和非公平锁，公平锁表示线程获取锁

的顺序是按照线程请求锁的时间早晚来决定的，也就是最早请求锁的 将最早获取到锁。

而非公平锁则在运行时闯入，也就是先来不一定先得。



ReentrantLock 提供了公平和非公平锁的实现。 构造函数不传递参数，默认为非公平锁。

2.12.3 独占锁和共享锁

![image-20200727224550701](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727224550701.png)

2.12.4 什么是可重入锁

![image-20200727224918482](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727224918482.png)

![image-20200727224926943](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727224926943.png)

2.12.5 自旋锁

![image-20200727225111859](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200727225111859.png)

## 第五章 CopyOnWriteArrayList

![image-20200729221222998](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200729221222998.png)

![image-20200729221141308](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200729221141308.png)

## 	第六章 锁的原理