# Springboot整合Dubbo实现远程服务调用

Dubbo是阿里巴巴开源的RPC框架，当服务拆分为多个服务时，服务与服务之间可以通过Dubbo实现远程服务调用

## 服务接口

新建一个`dubbo-interface`项目，里面存放着服务的接口

```java
/**
 * @author by cyf
 * @date 2020/10/4.
 */
public interface HelloService {

    public String sayHello(String name);
}
```

项目的结构

![image-20201009203524679](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20201009203524679.png)

使用`maven install`将其打包并传到本地仓库，供消费者和生产者使用。

## 服务生产者

建好`springboot`工程并引入依赖

```xml
 		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
		<!--刚才创建好的接口包-->
        <dependency>
            <groupId>com.cyf</groupId>
            <artifactId>dubbo-interface</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <!--引入dubbo的依赖-->
        <dependency>
            <groupId>com.alibaba.spring.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>
        <!--引入zookeeper-->
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.10</version>
        </dependency>
```

修改配置文件

```yaml
server:
  port: 8333

##dubbo 的注册中心选为zookeeper
spring:
  dubbo:
    application:
      name: dubbo-provider
      registry: zookeeper://localhost:2181
```

在主类上添加`@EnableDubboConfiguration`注解

```java
@SpringBootApplication
// 开启dubbo的自动配置
@EnableDubboConfiguration
public class DubboProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboProviderApplication.class, args);
    }

}

```

接口实现类

```java

/**
 * @author by cyf
 * @date 2020/10/4.
 */
@Service //此注解为dubbo提供的Service注解 高版本则为DubboService注解（等同于@Service+@Componet）
@Component
public class HelloServiceImpl implements HelloService {


    @Override
    public String sayHello(String name) {
        return "hello"+name;
    }
}

```

## 服务消费者

新建`springboot`工程并引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>com.cyf</groupId>
            <artifactId>dubbo-interface</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <!--引入dubbo的依赖-->
        <dependency>
            <groupId>com.alibaba.spring.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>
        <!--引入zookeeper-->
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.10</version>
        </dependency>
```

修改配置文件

```yml
server:
  port: 8444

spring:
  dubbo:
    application:
      name: dubbo-consumer
      registry: zookeeper://localhost:2181
```

在主类上添加`@EnableDubboConfiguration`注解



接口调用

```java

/**
 * @author by cyf
 * @date 2020/10/4.
 */
@RestController
public class HelloController {
    //@Reference 为接口实现代理类并提供远程调用
    @Reference
    private HelloService helloService;

    @RequestMapping("/hello")
    public String hello(){
        String hello = helloService.sayHello("cyf");
        System.out.println(helloService.sayHello("world"));
        return  hello;
    }
}

```



本地启动`zookeeper`端口为2181



访问`http://localhost:8444/hello` 即完成远程服务调用



![image-20201009204927813](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20201009204927813.png)