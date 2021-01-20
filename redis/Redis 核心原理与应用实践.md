# Redis 核心原理与应用实践

## 第一章 基础和应用篇

### 1.1 Redis 数据结构

Redis 中基础的数据结构

**分别是string list hash set zset**

**1、String**

```shell
String 基本命令

127.0.0.1:6379> set name cyf
OK
127.0.0.1:6379> get name
"cyf"
127.0.0.1:6379> exites name
(error) ERR unknown command 'exites'
127.0.0.1:6379> exists name kkk
(integer) 1
127.0.0.1:6379> exists name 
(integer) 1
127.0.0.1:6379> get name
"cyf"
127.0.0.1:6379> del name
(integer) 1
127.0.0.1:6379> mset name1 kkk name2 hhh
OK
127.0.0.1:6379> mget name1 name2
1) "kkk"
2) "hhh"

#设置过期时间
127.0.0.1:6379> expire name1 4
(integer) 1
127.0.0.1:6379> get name1
(nil)
127.0.0.1:6379> setex name1 2 hhh
OK
127.0.0.1:6379> get name1
(nil)
127.0.0.1:6379> set age 30
OK
127.0.0.1:6379> incr age  #自增长
(integer) 31
127.0.0.1:6379> incr age
(integer) 32
127.0.0.1:6379> incr age
(integer) 33

```

**2、list**
**list 相当于java里面的LinkedList 双向链表，底层是一个称为快速链表（quicklist）的一个结构**

```shell
#左进左出 相当于队列
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> llen books
(integer) 3
127.0.0.1:6379> lpop books
"python"
127.0.0.1:6379> lpop books
"java"
127.0.0.1:6379> 
127.0.0.1:6379> lpop books
"golang"
127.0.0.1:6379> lpop books
(nil)
#右进左出 相当于栈

127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> rpop books
"golang"
127.0.0.1:6379> rpop books
"java"
127.0.0.1:6379> rpop books
"python"
127.0.0.1:6379> rpop books
(nil)
```

**3、hash**

类似于HashMap



```shell
#HashMap中rehash是个耗时的操作，Redis为了追求性能，采用渐进式rehash策略
#渐进式rehash 会在rehash的同时，保留新旧两个hash结构，查询会同时查询两个hash结构，在后续会循序渐进地将旧hash的内容一点点迁移至新的hash结构中，当搬迁完成后，就使用新的hash
127.0.0.1:6379> hset books java "think in java"
(integer) 1
127.0.0.1:6379> hset books python "python book"
(integer) 1
127.0.0.1:6379> hset books golang "concurrency in go"
(integer) 1
127.0.0.1:6379> hgetall books
1) "java"
2) "think in java"
3) "python"
4) "python book"
5) "golang"
6) "concurrency in go"
127.0.0.1:6379> hlen books
(integer) 3
127.0.0.1:6379> hget books java
"think in java"
127.0.0.1:6379> hset books golang "learning go programming"
(integer) 0
127.0.0.1:6379> hget books golang
"learning go programming"
127.0.0.1:6379> hmset books java "learning java" python "hello python"
OK
127.0.0.1:6379> hgetall
(error) ERR wrong number of arguments for 'hgetall' command
127.0.0.1:6379> hgetall books
1) "java"
2) "learning java"
3) "python"
4) "hello python"
5) "golang"
6) "learning go programming"
127.0.0.1:6379> hincrby user-cyf age 24
(integer) 24
127.0.0.1:6379> hincrby user-cyf age 1
(integer) 25
127.0.0.1:6379> hincrby user-cyf age 1
(integer) 26
#### hash结构一般用来存储用户信息，可以对用户结构中的每个字段单独存储。当获取用户信息时可以进行部分获取
```

**4、set**

```shell
## 相当于HashSet  set可以用来去重，例如存储活动中奖id
127.0.0.1:6379> sadd books java
(integer) 1
127.0.0.1:6379> sadd books java
(integer) 0
127.0.0.1:6379> sadd books java python
(integer) 1
127.0.0.1:6379> smembers books
1) "python"
2) "java"
127.0.0.1:6379> sismember books java
(integer) 1
127.0.0.1:6379> sismember books hh
(integer) 0
127.0.0.1:6379> scard books
(integer) 2
127.0.0.1:6379> spop books
"python"
127.0.0.1:6379> spop books
"java"
127.0.0.1:6379> spop books
(nil)

```

**5、zset**

```shell
## zset是个有序的set 可以用来排序 内存存储信息分为value 和 score
127.0.0.1:6379> zadd books 9.0 "think in java"
(integer) 1
127.0.0.1:6379> zadd books 8.8 "java concurrency"
(integer) 1
127.0.0.1:6379> zadd books 7.0 "java cookbook"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "java concurrency"
3) "think in java"
127.0.0.1:6379> zrange books  0 -1
1) "java cookbook"
2) "java concurrency"
3) "think in java"
127.0.0.1:6379> zrevrange books 0 -1
1) "think in java"
2) "java concurrency"
3) "java cookbook"
127.0.0.1:6379> zcard books
(integer) 3
127.0.0.1:6379> zscore books "java cookbook:
Invalid argument(s)
127.0.0.1:6379> zscore books "java cookbook"
"7"
127.0.0.1:6379> zrank books "java cookbook"
(integer) 0
127.0.0.1:6379> zrangebyscore books 0 8.9
1) "java cookbook"
2) "java concurrency"
127.0.0.1:6379> zrem books "java cookbook"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java concurrency"
2) "think in java"

```

`zset`内部的排序功能是通过跳跃列表的 数据结构实现的。

### 1.2分布式锁

```shell
127.0.0.1:6379> set lock:codehole true ex 5 nx
OK
#do something...
127.0.0.1:6379> del lock:codehole
(integer) 1
```

会产生一个问题，当业务执行逻辑超过设定时间 锁会超时（所以适应在操作时间不长的场景）

可以在加锁前生成一个随机的`value` 解锁时判断是否是同一个`value`然后再删除key

但是判断和删除不是一个原子性操作

可以使用lua脚本保证原子性

### 1.3 HyperLogLog

HyperLogLog 数据结构是Redis的高级数据结构。

假如要去重统计一个网页的点击量。如果用set集合统计，当数据庞大时，去重功能会消耗很多的存储空间。

HyperLogLog 提供不精确的去重计数方法，标准误差是0.81%



```bash
pfadd #添加计数
pfcount #获取计数
pfmerge #将多个pf计数值累加形成新值
```

### 1.4什么是布隆过滤器

布隆过滤器是一个叫“布隆”的人提出的，它本身是一个很长的二进制向量，既然是二进制的向量，那么显而易见的，存放的不是0，就是1。

现在我们新建一个长度为16的布隆过滤器，默认值都是0，就像下面这样：
![image.png](https://upload-images.jianshu.io/upload_images/15100432-f5ef96d79638bc55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在需要添加一个数据：

我们通过某种计算方式，比如Hash1，计算出了Hash1(数据)=5，我们就把下标为5的格子改成1，就像下面这样：

![image.png](https://upload-images.jianshu.io/upload_images/15100432-0e9a0deac1c7e6f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们又通过某种计算方式，比如Hash2，计算出了Hash2(数据)=9，我们就把下标为9的格子改成1，就像下面这样：
![image.png](https://upload-images.jianshu.io/upload_images/15100432-2f5ba4726f822abf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还是通过某种计算方式，比如Hash3，计算出了Hash3(数据)=2，我们就把下标为2的格子改成1，就像下面这样：
![image.png](https://upload-images.jianshu.io/upload_images/15100432-9a0333841eacd242.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样，刚才添加的数据就占据了布隆过滤器“5”，“9”，“2”三个格子。

可以看出，仅仅从布隆过滤器本身而言，根本没有存放完整的数据，只是运用一系列随机映射函数计算出位置，然后填充二进制向量。

这有什么用呢？比如现在再给你一个数据，你要判断这个数据是否重复，你怎么做？

你只需利用上面的三种固定的计算方式，计算出这个数据占据哪些格子，然后看看这些格子里面放置的是否都是1，如果有一个格子不为1，那么就代表这个数字不在其中。这很好理解吧，比如现在又给你了刚才你添加进去的数据，你通过三种固定的计算方式，算出的结果肯定和上面的是一模一样的，也是占据了布隆过滤器“5”，“9”，“2”三个格子。

但是有一个问题需要注意，如果这些格子里面放置的都是1，不一定代表给定的数据一定重复，也许其他数据经过三种固定的计算方式算出来的结果也是相同的。这也很好理解吧，比如我们需要判断对象是否相等，是不可以仅仅判断他们的哈希值是否相等的。

也就是说布隆过滤器只能判断数据是否一定不存在，而无法判断数据是否一定存在。

按理来说，介绍完了新增、查询的流程，就要介绍删除的流程了，但是很遗憾的是布隆过滤器是很难做到删除数据的，为什么？你想想，比如你要删除刚才给你的数据，你把“5”，“9”，“2”三个格子都改成了0，但是可能其他的数据也映射到了“5”，“9”，“2”三个格子啊，这不就乱套了吗？

相信经过我这么一介绍，大家对布隆过滤器应该有一个浅显的认识了，至少你应该清楚布隆过滤器的优缺点了：

- 优点：由于存放的不是完整的数据，所以占用的内存很少，而且新增，查询速度够快；
- 缺点： 随着数据的增加，误判率随之增加；无法做到删除数据；只能判断数据是否一定不存在，而无法判断数据是否一定存在。

可以看到，布隆过滤器的优点和缺点一样明显。

在上文中，我举的例子二进制向量长度为16，由三个随机映射函数计算位置，在实际开发中，如果你要添加大量的数据，仅仅16位是远远不够的，为了让误判率降低，我们还可以用更多的随机映射函数、更长的二进制向量去计算位置。

### 1.5持久化

redis有两种持久化机制，一种是**快照**，另一种是**AOF日志**。

快照：快照是一次全量备份。

AOP日志： 是连续的增量备份。