# RabbitMQ-学习

## 1.什么是MQ

消息队列（Message Queue，简称MQ），从字面意思上看，本质是个队列，FIFO先入先出，只不过队列中存放的内容是message而已。
其主要用途：不同进程Process/线程Thread之间通信。
为什么会产生消息队列？有几个原因：

不同进程（process）之间传递消息时，两个进程之间耦合程度过高，改动一个进程，引发必须修改另一个进程，为了隔离这两个进程，在两进程间抽离出一层（一个模块），所有两进程之间传递的消息，都必须通过消息队列来传递，单独修改某一个进程，不会影响另一个；

不同进程（process）之间传递消息时，为了实现标准化，将消息的格式规范化了，并且，某一个进程接受的消息太多，一下子无法处理完，并且也有先后顺序，必须对收到的消息进行排队，因此诞生了事实上的消息队列；

## 2.学习五种队列

### 2.1简单队列

![image-20200521115943337](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521115943337.png)

P：消息的生产者
C：消息的消费者
红色：队列

#### 2.1.1 maven导入依赖

```xml
<dependencies>
        <!-- 引入队列依赖 -->
        <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>4.0.2</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.10</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.5</version>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
        </dependency>
    </dependencies>
```

#### 2.1.2获取MQ的连接

```java
/**rabbitMQ ConnectionUtil
 * @author by cyf
 * @date 2020/5/20.
 */
public class ConnectionUtil {

    public static  Connection getConnection() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        //设置账号密码  端口号 主机地址
        factory.setUsername("cyf");
        factory.setPassword("123");
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setVirtualHost("/mqtest");
        Connection connection = factory.newConnection();

        return connection;
    }
}
```

linux的rabbitMQ总是自动挂掉，采用本地rabbitMQ服务

#### 2.1.3生产者发送消息到队列

```java
/**
 * 生产者
 *
 * @author by cyf
 * @date 2020/5/20.
 */
public class Send {

    private static final String QUEUE_NAME = "test_simple_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();

        Channel channel = connection.createChannel();
        //创建队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        String message = "hello queue";
        System.out.println("message:" + message);
        //消息传入
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());

        channel.close();
        connection.close();
    }
}
```

#### 2.1.4消费者从队列中获取消息

```java
/**消费者
 * @author by cyf
 * @date 2020/5/20.
 */
public class Recv {
    private static final String QUEUE_NAME = "test_simple_queue";


    public static void main(String[] args) throws IOException, TimeoutException {
        //获取连接
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
		//消费者
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String smg = new String(body,"utf-8");
                System.out.println(new Date());
                System.out.println("recv:" + smg);
            }
        };
		//监听队列
        channel.basicConsume(QUEUE_NAME,defaultConsumer);
    }
```

### 2.2 Work模式

![image-20200521120449574](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521120449574.png)

一个生产者、2个消费者。

一个消息只能被一个消费者获取。

#### 2.2.1 生产者

```java
/**生产者
 * @author by cyf
 * @date 2020/5/21.
 */
public class Send {
    public static final String QUEUE_NAME = "work_queue";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //声明队列
        boolean durable = false;//消息持久化
        channel.queueDeclare(QUEUE_NAME,durable,false,false,null);

        for (int i = 0; i < 30; i++) {
            String msg = ""+i;
            //发送消息
            channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
            System.out.println("send:msg"+i);
            Thread.sleep(i*20);
        }

        channel.close();
        connection.close();
    }
}
```



#### 2.2.2 消费者1

```java
/**消费者1
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec1 {

    public static final String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[1] Recv:msg"+msg);
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[1] done");
                }
            }
        };
        Boolean autoAsk = true;
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }

}
```



#### 2.2.3 消费者2

```java
/**消费者2
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec2 {
    public static final String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //声明队列
        bol
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[2] Recv:msg"+msg);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[2] done");
                }
            }
        };
        Boolean autoAsk = true;
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}
```

2.24.测试
测试结果：
1、消费者1和消费者2获取到的消息内容是不同的，同一个消息只能被一个消费者获取。
2、消费者1和消费者2获取到的消息的数量是相同的，一个是消费奇数号消息，一个是偶数。

其实，这样是不合理的，因为消费者1线程停顿的时间短。应该是消费者1要比消费者2获取到的消息多才对。
RabbitMQ 默认将消息顺序发送给下一个消费者，这样，每个消费者会得到相同数量的消息。即轮询（round-robin）分发消息。

​	怎样才能做到按照每个消费者的能力分配消息呢？联合使用 Qos 和 Acknowledge 就可以做到。
basicQos 方法设置了当前信道最大预获取（prefetch）消息数量为1。消息从队列异步推送给消费者，消费者的 ack 也是异步发送给队列，从队列的视角去看，总是会有一批消息已推送但尚未获得 ack 确认，Qos 的 prefetchCount 参数就是用来限制这批未确认消息数量的。设为1时，队列只有在收到消费者发回的上一条消息 ack 确认后，才会向该消费者发送下一条消息。prefetchCount 的默认值为0，即没有限制，队列会将所有消息尽快发给消费者。

2个概念

**轮询分发** ：使用任务队列的优点之一就是可以轻易的并行工作。如果我们积压了好多工作，我们可以通过增加工作者（消费者）来解决这一问题，使得系统的伸缩性更加容易。在默认情况下，RabbitMQ将逐个发送消息到在序列中的下一个消费者(而不考虑每个任务的时长等等，且是提前一次性分配，并非一个一个分配)。平均每个消费者获得相同数量的消息。这种方式分发消息机制称为Round-Robin（轮询）。

**公平分发** ：虽然上面的分配法方式也还行，但是有个问题就是：比如：现在有2个消费者，所有的奇数的消息都是繁忙的，而偶数则是轻松的。按照轮询的方式，奇数的任务交给了第一个消费者，所以一直在忙个不停。偶数的任务交给另一个消费者，则立即完成任务，然后闲得不行。而RabbitMQ则是不了解这些的。这是因为当消息进入队列，RabbitMQ就会分派消息。它不看消费者为应答的数目，只是盲目的将消息发给轮询指定的消费者。

为了解决这个问题，我们使用basicQos( prefetchCount = 1)方法，来限制RabbitMQ只发不超过1条的消息给同一个消费者。当消息处理完毕后，有了反馈，才会进行第二次发送。
还有一点需要注意，使用公平分发，必须关闭自动应答，改为手动应答。

```java
//生产者里面添加        
//保证每次发生只发送一条给消费者
        int  prefetchCount = 1;
        channel.basicQos(prefetchCount);
```

```java
   //消费者里面添加     

		channel.basicQos(1);//保证一次只分发一个
        
               try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[2] done");
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        Boolean autoAsk = false;//自动应答  false 
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
```

消费者从队列中获取消息，服务端如何知道消息已经被消费呢？

### 2.3 消息应答与持久化

#### 2.3.1消息应答

**模式1：自动确认**
只要消息从队列中获取，无论消费者获取到消息后是否成功消息，都认为是消息已经成功消费。
**模式2：手动确认**
消费者从队列中获取消息后，服务器会将该消息标记为不可用状态，等待消费者的反馈，如果消费者一直没有反馈，那么该消息将一直处于不可用状态。

自动模式：

```java
boolean autoAsk = true;        
channel.basicConsume(QUEUE_NAME,autoAsk,consumer);//自动确认
```

手动确认

```
boolean autoAsk = false;        
channel.basicConsume(QUEUE_NAME,autoAsk,consumer);//手动确认
```

#### 2.3.2消息持久化

```java
        //声明队列
        boolean durable = false;//消息持久化
        channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
```

**可以将队列数据持久化，这样rabbitMQ挂了数据也不会丢失**

### 2.4 订阅模式



![image-20200521150030251](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521150030251.png)

**解读：**
1、1个生产者，多个消费者
2、每一个消费者都有自己的一个队列
3、生产者没有将消息直接发送到队列，而是发送到了交换机
4、每个队列都要绑定到交换机
5、生产者发送的消息，经过交换机，到达队列，实现，一个消息被多个消费者获取的目的
注意：一个消费者队列可以有多个消费者实例，只有其中一个消费者实例会消费

#### 2.4.1 消息的生产者（看作是后台系统）

向交换机中发送消息。

```java
/**生产者 订阅
 * @author by cyf
 * @date 2020/5/21.
 */
public class Send {
    public static final String EXCHANGE_NAME="exchange_fanout";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        //绑定交换机
        channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
        //消息
        String msg = "hello ps!";
        channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
        System.out.println("send:"+msg);

        channel.close();
        connection.close();

    }

}
```

**注意：消息发送到没有队列绑定的交换机时，消息将丢失，因为，交换机没有存储消息的能力，消息只能存在在队列中。**

#### 2.4.2 消息消费者

```java
/**消费者1 看作发向邮箱
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec1 {
    public static final String QUEUE_NAME = "queue_email";
    public static final String EXCHANGE_NAME ="exchange_fanout";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");

        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[1] Recv:msg"+msg);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[1] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}
```

```java
/**消费者2 看作发向手机
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec2 {
    public static final String QUEUE_NAME = "queue_phone";
    public static final String EXCHANGE_NAME ="exchange_fanout";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");

        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[2] Recv:msg"+msg);
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[2] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}
```

### 2.5 路由模式

![image-20200521154351921](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521154351921.png)

#### 2.5.1 生产者

```java
/**路由模式 生产者
 * @author by cyf
 * @date 2020/5/21.
 */
public class Send {

    public static final String EXCHANGE_NAME="exchange_direct";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        //绑定交换机， “direct”是交换机的类型，此处为路由模式
        channel.exchangeDeclare(EXCHANGE_NAME,"direct");
        String routingKey = "create";//key
        //消息
        String msg = "direct!";
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        System.out.println("send:"+msg);

        channel.close();
        connection.close();

    }
}
```

#### 2.5.2 消费者

```java
/**
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec1 {
    public static final String QUEUE_NAME = "queue_routing-1";
    public static final String EXCHANGE_NAME ="exchange_direct";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机，绑定两个key，分别为 delete和updatta
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"delete");
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"update");


        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[1] Recv:msg"+msg);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[1] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}
```

```java
/**
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec2 {
    public static final String QUEUE_NAME = "queue_routing-2";
    public static final String EXCHANGE_NAME ="exchange_direct";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机，绑定了三个key
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"delete");
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"update");
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"create");

        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[2] Recv:msg"+msg);
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[2] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}

```

### 2.6 主题模式

![image-20200521203043365](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521203043365.png)



![image-20200521202930365](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200521202930365.png)

类似于路由模式，交换机带有key

**匹配时 * 表示匹配一个**

​			**#表示匹配所有**

##### 2.6.1 生产者

```java
/**主题模式 生产者
 * @author by cyf
 * @date 2020/5/21.
 */
public class Send {
    public static final String EXCHANGE_NAME="exchange_topic";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();

        //绑定交换机
        channel.exchangeDeclare(EXCHANGE_NAME,"topic");
        //String routingKey = "goods.create";//key
        String routingKey = "goods.delete";
        //消息
        String msg = "商品";
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        System.out.println("send:"+msg);

        channel.close();
        connection.close();

    }


}
```

##### 2.6.2 消者者

```java
/**
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec1 {
    public static final String QUEUE_NAME = "queue_topic_1";
    public static final String EXCHANGE_NAME = "exchange_topic";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //将队列绑定到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.delete");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.update");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.insert");


        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body, "UTF-8");
                System.out.println("[1] Recv:msg:" + msg);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println("[1] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME, autoAsk, consumer);
    }
    
    
    
/**
 * @author by cyf
 * @date 2020/5/21.
 */
public class Rec2 {
    public static final String QUEUE_NAME = "queue_topic_2";
    public static final String EXCHANGE_NAME ="exchange_topic";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        final Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定到交换机
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"goods.#");//#表示匹配所有


        channel.basicQos(1);//保证一次只分发一个
        //消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String msg = new String(body,"UTF-8");
                System.out.println("[2] Recv:msg"+msg);
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("[2] done");
                    //每次消费完 通知队列继续发送
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //
        Boolean autoAsk = false;//自动应答  false
        channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
    }
}

```

## 3.springboot-rabbitMQ

添加依赖

```xmL
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

配置

```properties
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=cyf
spring.rabbitmq.password=123
spring.rabbitmq.virtual-host=/mqtest
```

配置类

```java
/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Configuration
public class RabbitConfig {

    @Bean
    public Queue queue(){
        return new Queue("q_hello");
    }
}
```

### 1、简单队列

生产者

```java
/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
public class Send {
    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send()  {
        String data = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        String msg = "hello rabbit" +data;
        System.out.println("Send:"+msg);
        //简单对列的情况下routingKey即为Q名
        rabbitTemplate.convertAndSend("q_hell",msg);

    }
```

消费者

```java
@Component
@RabbitListener(queues = "q_hello")
public class Receiver {

    @RabbitHandler
    public void process(String message){
        System.out.println(message);
    }
}
```

test

```java
@SpringBootTest
class DemoApplicationTests {

    @Autowired
    private Send sender;

    @Test
    void helloSimple() {
        sender.send();
    }
    
 输出：
     Send:hello rabbit2020-05-22 11:14:53
	 hello rabbit2020-05-22 11:14:53
```

### 2、Work模式

```java
/**生产者
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
public class Sender {


    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send(int i) {
        String data = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        String msg = "hello rabbit " + i +" "+ data;
        System.out.println("Send:" + msg);
        //简单对列的情况下routingKey即为Q名
        rabbitTemplate.convertAndSend("q_hello", msg);

    }
}

/**消费者1
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_hello")
public class Receiver {

    @RabbitHandler
    public void process(String message){
        System.out.println("Receiver1:"+message);
    }
}

/**消费者2
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_hello")
public class Receiver2 {

    @RabbitHandler
    public void process(String message){
        System.out.println("Receiver2:"+message);
    }
}
测试类
    
    @Autowired
    private Sender workSender;

 @Test
    void workQueue() throws InterruptedException {

        for (int i = 0; i <50 ; i++) {
            workSender.send(i);
            Thread.sleep(200);
        }
    }
输出：
Send:hello rabbit 0 2020-05-22 11:38:00
Receiver1:hello rabbit 0 2020-05-22 11:38:00
Send:hello rabbit 1 2020-05-22 11:38:00
Receiver2:hello rabbit 1 2020-05-22 11:38:00
Send:hello rabbit 2 2020-05-22 11:38:00
Receiver1:hello rabbit 2 2020-05-22 11:38:00
Send:hello rabbit 3 2020-05-22 11:38:01
Receiver2:hello rabbit 3 2020-05-22 11:38:01
Send:hello rabbit 4 2020-05-22 11:38:01
Receiver1:hello rabbit 4 2020-05-22 11:38:01
    ...
```

### 3、Topic Exchange（主题模式）

- topic 是RabbitMQ中最灵活的一种方式，可以根据routing_key自由的绑定不同的队列

首先对topic规则配置，这里使用两个队列(消费者)来演示。
1)配置队列，绑定交换机

```java
/**创建两个队列并绑定到同一个交换机，设置对应的routingkey
 * @author by cyf
 * @date 2020/5/22.
 */
@Configuration
public class TopicRabbitConfig {
    public static final String MESSAGE = "q_topic_message";
    public static final String MESSAGES = "q_topic_messages";
    //创建队列
    @Bean
    public Queue queueMessage(){
        return new Queue(MESSAGE);
    }

    @Bean
    public Queue queueMessages(){
        return new Queue(MESSAGES);
    }
    //声明topic交换机
    @Bean
    public TopicExchange exchange(){
        return new TopicExchange("topicExchange");
    }
    //将队列绑定到交换机，并指定routingkey
    @Bean
    public Binding  bingExchangeMessage(Queue queueMessage,TopicExchange exchange){
        return  BindingBuilder.bind(queueMessage).to(exchange).with("topic.message");
    }
    @Bean
    public Binding  bingExchangeMessages(Queue queueMessages,TopicExchange exchange){
        return BindingBuilder.bind(queueMessages).to(exchange).with("topic.#");
    }
}
```

**生产者** **和两个消费者**

```java
/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
public class MagSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;
    public void Send1(){
        String message = "hello,i am message1";
        System.out.println("send:"+message);
        rabbitTemplate.convertAndSend("topicExchange","topic.message",message);
    }

    public void Send2(){
        String message = "hello,i am message2";
        System.out.println("send:"+message);
        rabbitTemplate.convertAndSend("topicExchange","topic.messages",message);
    }
}


/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_topic_message")
public class TopicReceiver1 {

    @RabbitHandler
    public void receiver(String message){
        System.out.println("Receiver1:"+message);
    }
}

/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_topic_messages")
public class TopicReceiver2 {

    @RabbitHandler
    public void receiver(String message){
        System.out.println("Receiver2:"+message);
    }
}

```

send1方法会匹配到topic.#和topic.message，

两个Receiver都可以收到消息，

发送send2只有topic.#可以匹配所有只有Receiver2监听到消息。



测试

```java
    @Test
    void topicQueue(){
        magSender.Send1();
        magSender.Send2();
    }
 输出：  
send:hello,i am message1
send:hello,i am message2
Receiver1:hello,i am message1
Receiver2:hello,i am message1
Receiver2:hello,i am message2
```

### 4、订阅模式

- Fanout 就是我们熟悉的广播模式或者订阅模式，给Fanout交换机发送消息，绑定了这个交换机的所有队列都收到这个消息。

  1、配置队列，绑定交换机

```java
/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Configuration
public class FanoutRabbitConfig {

    @Bean
    public Queue aMessage() {
        return new Queue("q_fanout_A");
    }

    @Bean
    public Queue bMessage() {
        return new Queue("q_fanout_B");
    }

    @Bean
    public Queue cMessage() {
        return new Queue("q_fanout_C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("mybootfanoutExchange");
    }

    @Bean
    Binding bindingExchangeA(Queue aMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(aMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeB(Queue bMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(bMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeC(Queue cMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(cMessage).to(fanoutExchange);
    }

}
```

2、**生产者 消费者**

```java
@Component
public class MySend {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send() {
        String context = "hi, fanout msg ";
        System.out.println("Sender : " + context);
        this.rabbitTemplate.convertAndSend("mybootfanoutExchange","", context);
    }

}

/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_fanout_A")
public class FanoutRec1 {
    @RabbitHandler
    public void process(String hello) {
        System.out.println("AReceiver  : " + hello + "/n");
    }
}


/**
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "q_fanout_B")
public class FanoutRec2 {
    @RabbitHandler
    public void process(String hello) {
        System.out.println("BReceiver  : " + hello + "/n");
    }
}


    @Test
    void fanoutQueue(){
        mySend.send();
    }
输出：
Sender : hi, fanout msg 
AReceiver  : hi, fanout msg /n
BReceiver  : hi, fanout msg /n
```

