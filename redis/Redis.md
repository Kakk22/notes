# Docker 安装 Redis

### **1. 拉取镜像**

```
docker pull redis
```

### 2. 查看镜像

```
docker images
```

### 3.配置文件

首先创建一个redis的目录

```
mkdir /usr/local/redis
```

下载配置文件到指定目录

```
wget -P  /usr/local/redis http://download.redis.io/redis-stable/redis.conf
```

修改配置文件

```
vi /usr/local/redis/redis.conf
```

![img](https://img2020.cnblogs.com/blog/711069/202003/711069-20200312123241748-985442449.gif)

1. 进入文件命令模式后，输入‘**/bind 127.0.0.1**’查找关键字找到对应的行，输入‘**N**’查找下一个
2. 找到所在行后输入字母‘**i**’ 进入编辑模式，在第一位字母前面加上‘**#**’注释行。
3. 关闭保护模式：protected-mode no
4. 按‘ESC’进入命令行模式输入‘**:wq**’ 保存文件并退出，完成配置文件的编辑

| 配置           | 说明                                           |
| -------------- | ---------------------------------------------- |
| bind 127.0.0.1 | 限制redis只能本地访问，#注释后所有Ip都可以访问 |

### 4.启动容器

```
docker run -d --name redis -p 6379:6379 -v /usr/local/redis/redis.conf:/etc/redis/redis.conf redis redis-server /etc/redis/redis.conf --appendonly yes
```

| 参数                                                 | 说明                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| -d                                                   | 后台运行容器                                                 |
| --name redis                                         | 设置容器名为 redis                                           |
| -p 6379:6379                                         | 将宿主机 6379 端口映射到容器的 6379 端口                     |
| -v /usr/local/redis/redis.conf:/etc/redis/redis.conf | 将宿主机redis.conf映射到容器内                               |
| -v /usr/local/redis/data:/data                       | 将宿主机 /usr/local/redis/data 映射到容器 /data , 方便备份持久数据 |
| redis-server /etc/redis/redis.conf                   | redis 服务以容器内的 redis.conf 配置启动                     |
| --appendonly yes                                     | redis 数据持久化                                             |

# 2 Redis 命令

### 1、docker 启动 redis客户端 

 redis-cli 表示运行一个redis客户端。

```shell
[root@localhost ~]# docker exec -it da45019bf760 redis-cli
127.0.0.1:6379> 
127.0.0.1:6379> set msg "Hello World Redis"
OK
127.0.0.1:6379> get msg
"Hello World Redis"
127.0.0.1:6379>
```

### 2、Redis常用命令

Redis五种数据类型：string、hash、list、set、zset

#### 2.1**公用命令**

```
DEL key
DUMP key：序列化给定key，返回被序列化的值
EXISTS key：检查key是否存在
EXPIRE key second：为key设定过期时间
TTL key：返回key剩余时间
PERSIST key：移除key的过期时间，key将持久保存
KEY pattern：查询所有符号给定模式的key
RANDOM key：随机返回一个key
RANAME key newkey：修改key的名称
MOVE key db：移动key至指定数据库中
TYPE key：返回key所储存的值的类型
```



**EXPIRE key second的使用场景**： 

1、限时的优惠活动

 2、网站数据缓存

 3、手机验证码 

4、限制网站访客频率



### 3、String类型

string类型是二进制安全的，redis的string可以包含任何数据，如图像、序列化对象。一个键最多能存储512MB。==二进制安全是指，在传输数据的时候，能保证二进制数据的信息安全，也就是不会被篡改、破译；如果被攻击，能够及时检测出来 ==

#### 3.1常用命令

```
setkey_name value：命令不区分大小写，但是key_name区分大小写
SETNX key value：当key不存在时设置key的值。（SET if Not eXists）
get key_name
GETRANGE key start end：获取key中字符串的子字符串，从start开始，end结束
MGET key1 [key2 …]：获取多个key
GETSET KEY_NAME VALUE：设定key的值，并返回key的旧值。当key不存在，返回nil
STRLEN key：返回key所存储的字符串的长度
INCR KEY_NAME ：INCR命令key中存储的值+1,如果不存在key，则key中的值话先被初始化为0再加1
INCRBY KEY_NAME 增量
DECR KEY_NAME：key中的值自减一
DECRBY KEY_NAME
append key_name value：字符串拼接，追加至末尾，如果不存在，为其赋值
```



### 4、hash类型

Redis hash是一个string类型的field和value的映射表，hash特别适用于存储对象。每个hash可以存储232-1键值对。可以看成KEY和VALUE的MAP容器。相比于JSON，hash占用很少的内存空间。

#### **4.1常用命令**

```shell
HSET key_name field value：为指定的key设定field和value
hmset key field value[field1,value1]
hget key field
hmget key field[field1]
hgetall key：返回hash表中所有字段和值
hkeys key：获取hash表所有字段
hlen key：获取hash表中的字段数量
-hdel key field [field1]：删除一个或多个hash表的字段
```



#### **4.2应用场景**

Hash的应用场景，通常用来存储一个用户信息的对象数据。
1、相比于存储对象的string类型的json串，json串修改单个属性需要将整个值取出来。而hash不需要。
2、相比于多个key-value存储对象，hash节省了很多内存空间
3、如果hash的属性值被删除完，那么hash的key也会被redis删除



### 5、登录限制功能练习

业务逻辑

![image-20200328170400132](C:\Users\Cheny\AppData\Roaming\Typora\typora-user-images\image-20200328170400132.png)

### 6、List 类型

list
类似于Java中的LinkedList。

#### 6.1常用命令

```shell
lpush key value1 [value2]
rpush key value1 [value2]
lpushx key value：从左侧插入值，如果list不存在，则不操作
rpushx key value：从右侧插入值，如果list不存在，则不操作
llen key：获取列表长度
lindex key index：获取指定索引的元素
lrange key start stop：获取列表指定范围的元素
lpop key ：从左侧移除第一个元素
prop key：移除列表最后一个元素
blpop key [key1] timeout：移除并获取列表第一个元素，如果列表没有元素会阻塞列表到等待超时或发现可弹出元素为止
brpop key [key1] timeout：移除并获取列表最后一个元素，如果列表没有元素会阻塞列表到等待超时或发现可弹出元素为止
ltrim key start stop ：对列表进行修改，让列表只保留指定区间的元素，不在指定区间的元素就会被删除
lset key index value ：指定索引的值
linsert key before|after world value：在列表元素前或则后插入元素
```



#### **6.2应用场景**

1、对数据大的集合数据删减
		列表显示、关注列表、粉丝列表、留言评价...分页、热点新闻等
2、任务队列
		list通常用来实现一个消息队列，而且可以确保先后顺序，不必像MySQL那样通过order by来排序

**rpoplpush list1 list2**:  移除list1最后一个元素，并将该元素添加到list2并返回此元素
用此命令可以实现订单下单流程、用户系统登录注册短信等



### 7、set

**唯一、无序**

#### 7.1 常用命令

```shell
sadd key value1[value2]：向集合添加成员
scard key：返回集合成员数
smembers key：返回集合中所有成员
sismember key member：判断memeber元素是否是集合key成员的成员
srandmember key [count]：返回集合中一个或多个随机数
srem key member1 [member2]：移除集合中一个或多个成员
spop key：移除并返回集合中的一个随机元素
smove source destination member：将member元素从source集合移动到destination集合
sdiff key1 [key2]：返回所有集合的差集
sdiffstore destination key1[key2]：返回给定所有集合的差集并存储在destination中
sinter key1 [key2]:返回给定集合的交集
sinterstore destination key1[key2] :返回给定所有集合的交集并存储在destination中
sunion key1 [key2]:返回给定集合的并集
sunionstore destination key1 [key2] :返回给定所有集合的并集并存储在destination中
```

#### 7.2 应用场景 

对两个集合间的数据[计算]进行交集、并集、差集运算 1、以非常方便的实现如共同关注、共同喜好、二度好友等功能。对上面的所有集合操作，你还可以使用不同的命令选择将结果返回给客户端还是存储到一个新的集合中。 2、利用唯一性，可以统计访问网站的所有独立 IP

### 8、zset

**有序且不重复。每个元素都会关联一个double类型的分数，Redis通过分数进行从小到大的排序。分数可以重复**

#### 8.1常用命令

```shell
ZADD key score1 memeber1
ZCARD key ：获取集合中的元素数量
ZCOUNT key min max 计算在有序集合中指定区间分数的成员数
ZRANK key member：返回有序集合指定成员的索引
ZREVRANGE key start stop ：返回有序集中指定区间内的成员，通过索引，分数从高到底
ZREM key member [member …] 移除有序集合中的一个或多个成员
ZREMRANGEBYRANK key start stop 移除有序集合中给定的排名区间的所有成员(第一名是0)(低到高排序）
ZREMRANGEBYSCORE key min max 移除有序集合中给定的分数区间的所有成员
```

#### 8.2常用场景

1、常用于排行榜：
2、如推特可以以发表时间作为score来存储
3、存储成绩
4、还可以用zset来做带权重的队列，让重要的任务先执行

# 3、Redis功能特性

### 1、发布订阅

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis 客户端可以订阅任意数量的频道。

下图展示了频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系：

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

### 2、命令

```shell
subscribe channel [channel…]：订阅一个或多个频道的信息
psubscribe pattern [pattern…]：订阅一个或多个符合规定模式的频道
publish channel message ：将信息发送到指定频道
unsubscribe [channel[channel…]]：退订频道
punsubscribe [pattern[pattern…]]：退订所有给定模式的频道
```

### 3、应用场景

构建实时的消息系统，比如普通聊天、群聊等功能。
1、博客网站订阅，当作者发布就可以推送给粉丝
2、微信公众号模式

# 4、多数据库



```
- select db
- move key db
- flushdb
- flushall


```

# 5、Redis 事务

Redis事务可以一次执行多个命令，（按顺序地串行化执行，执行过程中不允许其他命令插入执行序列中）。
1、Redis会将一个事务中的所有命令序列化，然后按顺序执行
2、执行中不会被其他命令插入，不允许加塞行为

## 5.1命令

![命令](https://img-blog.csdnimg.cn/20190925163101930.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)

1. 输入Multi命令开始，输入的命令都会依次进入命令队列中，但不会执行
2. 直到输入exec后，Redis会将之前队列中的命令依次执行
3. discard放弃队列执行
4. 如果某个命令报出错，则只有报错的命令不会被执行，而其他的命令都会执行，不会回滚。
5. 如果队列中某个命令出现报告错误（语法错误），执行时整个队列都会被取消

```shell
127.0.0.1:6379> multi
OK
127.0.0.1:6379> get account:1
QUEUED
127.0.0.1:6379> get account:2
QUEUED
127.0.0.1:6379> decrby account:1 20
QUEUED
127.0.0.1:6379> incrby account:2 20
QUEUED
127.0.0.1:6379> get account:1
QUEUED
127.0.0.1:6379> get account:2
QUEUED
127.0.0.1:6379> exec
1) "90"
2) "80"
3) (integer) 70
4) (integer) 100
5) "70"
6) "100"
```

# 6、Redis持久化

### 6.1 RDB

RDB是Redis默认持久化机制。RDB相当于快照，保存的是一种状态

优点：
保存速度、还原速度极快
适用于灾难备份
缺点：
小内存的机器不符合使用。RDB机制符合要求就会快照。

### 6.2 AOF

如果Redis意外down掉，RDB方式会丢失最后一次快照后的所有修改。如果要求应用不能丢失任何修改，可以采用AOF持久化方式。
**AOF**：Append-Only File：Redis会将没一个收到写命令都追加到文件中（默认是appendonly.aof）。当Redis重启时会通过重新执行文件中的写命令重建整个数据库的内容。

产生的问题：
有些命令是多余的。

# 7、主从复制

什么是主从复制
持久化保证了即使 redis 服务重启也会丢失数据，因为 redis 服务重启后会将硬盘上持久化的数据恢复到内存中，但是当 redis 服务器的硬盘损坏了可能会导致数据丢失，如果通过 redis 的主从复制机制就可以避免这种单点故障，如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092615004529.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)


说明：

主 redis 中的数据有两个副本（replication）即从 redis1 和从 redis2，即使一台 redis 服务器宕机其它两台 redis 服务也可以继续提供服务。

主 redis 中的数据和从 redis 上的数据保持实时同步，当主 redis 写入数据时通过主从复制机制会复制到两个从 redis 服务上。

只有一个主 redis，可以有多个从 redis。

主从复制不会阻塞 master，在同步数据时，master 可以继续处理 client 请求。

一个 redis 可以即是主又是从，如下图：



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926150149596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)

主从复制配置
1.复制一份redis.conf，并修改redis.conf
cp /usr/local/redis/redis.conf /usr/local/dockerData/redis.conf
修改redis.conf，添加slaveof

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926144442306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDIzNDE4,size_16,color_FFFFFF,t_70)


slaveof master的ip master的port
确保master已开启

如果master有密码，那么需要设置在slave中设置master密码：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190926210306384.png)

**docker启动redis服务**

```dockerfile
docker run -it -p 6380:6380 
-v /usr/local/dockerData/redis.conf:/usr/local/etc/redis/redis.conf 
-v /usr/local/usrData:/data/:rw 
-- name myredis
--privileged=true redis 
redis-server /usr/local/etc/redis/redis.conf
```


privileged=true #给容器添加权限


privileged=true #给容器添加权限

执行两次即可创建两个slave，要注意映射的端口和name都要不一致。

验证主从复制

![image-20200330142630660](C:\Users\Cheny\AppData\Roaming\Typora\typora-user-images\image-20200330142630660.png)

![image-20200330142704156](C:\Users\Cheny\AppData\Roaming\Typora\typora-user-images\image-20200330142704156.png)

# 8、集群