# mall整合RabbitMQ实现延迟消息

## 业务场景说明

> 用于解决用户下单以后，订单超时如何取消订单的问题。

- 用户进行下单操作（会有锁定商品库存、使用优惠券、积分一系列的操作）；
- 生成订单，获取订单的id；
- 获取到设置的订单超时时间（假设设置的为60分钟不支付取消订单）；
- 按订单超时时间发送一个延迟消息给RabbitMQ，让它在订单超时后触发取消订单的操作；
- 如果用户没有支付，进行取消订单操作（释放锁定商品库存、返还优惠券、返回积分一系列操作）。

## 整合RabbitMQ实现延迟队列

### 在pom.xml中添加相关依赖

```xml
<!--消息队列相关依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<!--lombok依赖-->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### 修改SpringBoot配置文件

> 修改`application.yml`文件，在spring节点下添加`RabbitMQ`相关配置。

```yml
  rabbitmq:
    host: localhost # rabbitmq的连接地址
    port: 5672 # rabbitmq的连接端口号
    virtual-host: /myMall # rabbitmq的虚拟host
    username: cyf # rabbitmq的用户名
    password: 123 # rabbitmq的密码
    publisher-confirms: true #如果对异步消息需要回调必须设置为true
```

### 修改SpringBoot配置文件

> 修改`application.yml`文件，在spring节点下添加`RabbitMQ`相关配置。

```yml
  rabbitmq:
    host: localhost # rabbitmq的连接地址
    port: 5672 # rabbitmq的连接端口号
    virtual-host: /mall # rabbitmq的虚拟host
    username: mall # rabbitmq的用户名
    password: mall # rabbitmq的密码
    publisher-confirms: true #如果对异步消息需要回调必须设置为trueCopy to clipboardErrorCopied
```

### 添加消息队列的枚举配置类QueueEnum

> 用于延迟消息队列及处理取消订单消息队列的常量定义，包括交换机名称、队列名称、路由键名称。

```java
package com.cyf.malldemo.dto;

import lombok.Getter;

/**
 * 消息队列枚举配置
 *
 * @author by cyf
 * @date 2020/5/22.
 */
@Getter
public enum QueueEnum {
    /**
     * 消息通知队列
     */
    QUEUE_ORDER_CANCEL("mall.order.direct","mall.order.cancel","mall.order.cancel"),
    /**
     * 消息通知ttl
     */
    QUEUE_TTL_ORDER_CANCEL("mall.order.direct.ttl","mall.order.cancel.ttl","mall.order.cancel.ttl");
    /**
     * 交换名称
     */
    private String exchange;
    /**
     * 队列名字
     */
    private String name;
    /**
     * 路由键
     */
    private String routekey;

    QueueEnum(String exchange, String name, String routekey) {
        this.exchange = exchange;
        this.name = name;
        this.routekey = routekey;
    }
}

```

### 添加RabbitMQ的配置

> 用于配置交换机、队列及队列与交换机的绑定关系。

```java
package com.cyf.malldemo.config;

import com.cyf.malldemo.dto.QueueEnum;
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 消息队列配置
 *
 * @author by cyf
 * @date 2020/5/22.
 */
@Configuration
public class RabbitMqConfig {

    /**
     * 订单消息实际消费队列所绑定的交换机
     *
     * @return
     */
    @Bean
    DirectExchange orderDirect() {
        return (DirectExchange) ExchangeBuilder
                .directExchange(QueueEnum.QUEUE_ORDER_CANCEL.getExchange())
                .durable(true)
                .build();
    }

    /**
     * 订单延迟队列所绑定的交换机
     *
     * @return
     */
    @Bean
    DirectExchange orderTtlDirect() {
        return ExchangeBuilder
                .directExchange(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange())
                .durable(true)
                .build();
    }

    /**
     * 订单队列
     *
     * @return
     */
    @Bean
    public Queue orderQueue() {
        return new Queue(QueueEnum.QUEUE_ORDER_CANCEL.getName());
    }

    /**
     * 订单延迟队列（死信队列）
     *
     * @return
     */
    @Bean
    public Queue orderTtlQueue() {
        return QueueBuilder
                .durable(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getName())
                .deadLetterExchange(QueueEnum.QUEUE_ORDER_CANCEL.getExchange())//死后转发的交换机
                .deadLetterRoutingKey(QueueEnum.QUEUE_ORDER_CANCEL.getRoutekey())//死后转发的routekey
                .build();
    }

    /**
     * 将订单队列绑定到交换机
     *
     * @param orderQueue
     * @param orderDirect
     * @return
     */
    @Bean
    public Binding orderBinding(Queue orderQueue, DirectExchange orderDirect) {
        return BindingBuilder
                .bind(orderQueue)
                .to(orderDirect)
                .with(QueueEnum.QUEUE_ORDER_CANCEL.getRoutekey());
    }

    /**
     * 将订单队列绑定到交换机
     *
     * @param orderTtlQueue
     * @param orderTtlDirect
     * @return
     */
    @Bean
    public Binding orderTtlBinding(Queue orderTtlQueue, DirectExchange orderTtlDirect) {
        return BindingBuilder
                .bind(orderTtlQueue)
                .to(orderTtlDirect)
                .with(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRoutekey());
    }


}

```

#### 交换机及队列说明

- `mall.order.direct`（取消订单消息队列所绑定的交换机）:绑定的队列为`mall.order.cancel`，一旦有消息以`mall.order.cancel`为路由键发过来，会发送到此队列。
- `mall.order.direct.ttl`（订单延迟消息队列所绑定的交换机）:绑定的队列为`mall.order.cancel.ttl`，一旦有消息以`mall.order.cancel.ttl`为路由键发送过来，会转发到此队列，并在此队列保存一定时间，等到超时后会自动将消息发送到`mall.order.cancel`（取消订单消息消费队列）。

### 添加延迟消息的发送者CancelOrderSender

> 用于向订单延迟消息队列`mall.order.cancel.ttl`里发送消息。

```java
package com.cyf.malldemo.component;

import com.cyf.malldemo.dto.QueueEnum;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.AmqpException;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;


/**取消订单发送者
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
public class CancelOrderSender {

    private static final Logger LOGGER = LoggerFactory.getLogger(CancelOrderSender.class);

    @Autowired
    private AmqpTemplate amqpTemplate;

    // TODO: 2020/5/22 给延迟队列发送消息 
    public void sendMessage(Long orderId,final Long delayTimes){
        amqpTemplate.convertAndSend(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(),
                QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRoutekey(),
                orderId, new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        //给消息设置延迟毫秒值
                        message.getMessageProperties().setExpiration(String.valueOf(delayTimes));
                        return message;
                    }
                });
        LOGGER.info("send delay message orderId {}" ,orderId);
    }


}

```

### 添加取消订单消息的接收者CancelOrderReceiver

> 用于从取消订单的消息队列`mall.order.cancel`里接收消息。

```java
package com.cyf.malldemo.component;

import com.cyf.malldemo.service.OmsPortalOrderService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**订单取消接受者
 * @author by cyf
 * @date 2020/5/22.
 */
@Component
@RabbitListener(queues = "mall.order.cancel")
public class CancelOrderReceiver {

    private static final Logger LOGGER = LoggerFactory.getLogger(CancelOrderReceiver.class);

    @Autowired
    private OmsPortalOrderService portalOrderService;

    /**
     * 接收到消息后执行取消订单
     * @param orderId 订单id
     */
    @RabbitHandler
    public void handle(Long orderId){
        portalOrderService.cancelOrder(orderId);
        LOGGER.info("receive delay message orderId:{}",orderId);
    }
}

```



```java
package com.cyf.malldemo.service;

import com.cyf.malldemo.common.CommonResult;
import com.cyf.malldemo.dto.OrderParam;
import org.springframework.transaction.annotation.Transactional;

/**前台订单管理
 * @author by cyf
 * @date 2020/5/22.
 */
public interface OmsPortalOrderService {
    /**
     * 根据提交信息生成订单
     */
    @Transactional
    CommonResult generateOrder(OrderParam orderParam);

    /**
     * 取消单个超时订单
     */
    @Transactional
    void cancelOrder(Long orderId);
}

```



```java
package com.cyf.malldemo.service.impl;

import com.cyf.malldemo.common.CommonResult;
import com.cyf.malldemo.component.CancelOrderSender;
import com.cyf.malldemo.dto.OrderParam;
import com.cyf.malldemo.service.OmsPortalOrderService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 前台订单管理
 *
 * @author by cyf
 * @date 2020/5/22.
 */
@Service
public class OmsPortalOrderServiceImpl implements OmsPortalOrderService {
    private static Logger LOGGER = LoggerFactory.getLogger(OmsPortalOrderServiceImpl.class);
    @Autowired
    private CancelOrderSender cancelOrderSender;

    @Override
    public CommonResult generateOrder(OrderParam orderParam) {

        // TODO: 2020/5/22 执行一系列下单操作
        LOGGER.info("process generateOrder");
        //下单完成后，开启一个延迟消息，当用户没有付款操过一定时间时取消订单(orderId在下单后生成)
        sendDelayMessageCancelOrder(11L);
        return CommonResult.success(null,"下单成功");
    }


    @Override
    public void cancelOrder(Long orderId) {
		//查看订单是否付款，未付款执行取消操作
    }

    //设置订单取消时间
    private void sendDelayMessageCancelOrder(long orderId) {
        //设置超时时间,测试设置30秒
        Long delayTimes = 30 * 1000L;
        //发送延迟消息
        cancelOrderSender.sendMessage(orderId,delayTimes);

    }
}

```

```java
package com.cyf.malldemo.controller;

import com.cyf.malldemo.common.CommonResult;
import com.cyf.malldemo.dto.OrderParam;
import com.cyf.malldemo.service.OmsPortalOrderService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author by cyf
 * @date 2020/5/22.
 */
@RestController
@Api(tags = "OmsPortalOrderController", description = "订单管理")
@RequestMapping("/order")
public class OmsPortalOrderController {
    @Autowired
    private OmsPortalOrderService omsPortalOrderService;

    @ApiOperation("根据购物车信息生产订单")
    @RequestMapping(value = "/generateOrder",method = RequestMethod.POST)
    public CommonResult generateOrder(@RequestBody OrderParam orderParam){
        return omsPortalOrderService.generateOrder(orderParam);
    }
}

```

## 进行接口测试

### 调用下单接口

已经将延迟消息时间设置为30秒

![image-20200525104059784](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200525104059784.png)

![image-20200525104108295](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200525104108295.png)

![image-20200525104326137](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200525104326137.png)