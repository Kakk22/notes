# 使用docker compose 搭建Redis Cluster集群

## Redis集群搭建

- 在搭建Redis集群之前，我们需要修改下Redis的配置文件`redis.conf`，该文件的下载地址：https://github.com/antirez/redis/blob/5.0/redis.conf
- 需要修改的属性如下，主要是修改了一些集群配置和运行端口，端口号需要按需修改为6391~6396：

```bash
# 开启集群功能
cluster-enabled yes
# 设置运行端口
port 6391
# 设置节点超时时间，单位毫秒
cluster-node-timeout 15000
# 集群内部配置文件
cluster-config-file "nodes-6391.conf"
##在dokcer容器中部署要添加以下配置以保证节点与节点之间能够完成通信
# redis运行端口
cluster-announce-port 6391
# 云服务器上部署需指定公网ip
cluster-announce-ip 公网ip
# Redis总线端口，用于与其它节点通信
cluster-announce-bus-port 16391
```

编写docker-compose.yml 文件编排6个Redis容器

```yaml
version: "3"
services:
  redis-master1:
    image: redis:5.0 # 基础镜像
    container_name: redis-master1 # 容器名称
    working_dir: /config # 切换工作目录
    environment: # 环境变量
      - PORT=6391 # 会使用config/nodes-${PORT}.conf这个配置文件
    ports: # 映射端口，对外提供服务
      - 6391:6391 # redis的服务端口
      - 16391:16391 # redis集群监控端口
    stdin_open: true # 标准输入打开
    tty: true # 后台运行不退出
    network_mode: host # 使用host模式
    privileged: true # 拥有容器内命令执行的权限
    volumes:
      - /mydata/redis-cluster/config:/config #配置文件目录映射到宿主机
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint: # 设置服务默认的启动程序
      - /bin/bash
      - redis.sh
  redis-master2:
    image: redis:5.0
    working_dir: /config
    container_name: redis-master2
    environment:
      - PORT=6392
    ports:
      - 6392:6392
      - 16392:16392
    stdin_open: true
    network_mode: host
    tty: true
    privileged: true
    volumes:
      - /mydata/redis-cluster/config:/config
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint:
      - /bin/bash
      - redis.sh
  redis-master3:
    image: redis:5.0
    container_name: redis-master3
    working_dir: /config
    environment:
      - PORT=6393
    ports:
      - 6393:6393
      - 16393:16393
    stdin_open: true
    network_mode: host
    tty: true
    privileged: true
    volumes:
      - /mydata/redis-cluster/config:/config
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint:
      - /bin/bash
      - redis.sh
  redis-slave1:
    image: redis:5.0
    container_name: redis-slave1
    working_dir: /config
    environment:
      - PORT=6394
    ports:
      - 6394:6394
      - 16394:16394
    stdin_open: true
    network_mode: host
    tty: true
    privileged: true
    volumes:
      - /mydata/redis-cluster/config:/config
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint:
      - /bin/bash
      - redis.sh
  redis-slave2:
    image: redis:5.0
    working_dir: /config
    container_name: redis-slave2
    environment:
      - PORT=6395
    ports:
      - 6395:6395
      - 16395:16395
    stdin_open: true
    network_mode: host
    tty: true
    privileged: true
    volumes:
      - /mydata/redis-cluster/config:/config
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint:
      - /bin/bash
      - redis.sh
  redis-slave3:
    image: redis:5.0
    container_name: redis-slave3
    working_dir: /config
    environment:
      - PORT=6396
    ports:
      - 6396:6396
      - 16396:16396
    stdin_open: true
    network_mode: host
    tty: true
    privileged: true
    volumes:
      - /mydata/redis-cluster/config:/config
      - /mydata/redis-cluster/data:/data #数据目录映射到宿主机
    entrypoint:
      - /bin/bash
      - redis.sh

```

- 从docker-compose.yml文件中我们可以看到，我们的Redis容器分别运行在6391~6396这6个端口之上， 将容器中的`/config`配置目录映射到了宿主机的`/mydata/redis-cluster/config`目录，同时还以`redis.sh`脚本作为该容器的启动脚本；
- `redis.sh`脚本的作用是根据environment环境变量中的`PORT`属性，以指定配置文件来启动Redis容器；

```bash
redis-server  /config/nodes-${PORT}.conf
```

- 接下来我们需要把Redis的配置文件和`redis.sh`上传到Linux服务器的`/mydata/redis-cluster/config`目录下；

![image-20200926115042010](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200926115042010.png)

- 接下来上传我们的docker-compose.yml文件到Linux服务器，并使用docker-compose命令来启动所有容器；

![image-20200926115328731](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200926115328731.png)

- 此时进入其中一个Redis容器之中，初始化Redis集群；

```bash
# 进入Redis容器
docker exec -it redis-master1 /bin/bash
# 初始化Redis集群命令
redis-cli --cluster create \
47.107.53.172:6391 47.107.53.172:6392 47.107.53.172:6393 \
47.107.53.172:6394 47.107.53.172:6395 47.107.53.172:6396 \
--cluster-replicas 1 ##主节点和从节点比例为1
```

- 创建成功后我们可以使用`redis-cli`命令连接到其中一个Redis服务；

```bash
# 单机模式启动
redis-cli -h 127.0.0.1 -p 6391
# 集群模式启动
redis-cli -c -h 127.0.0.1 -p 6391
```

- 之后通过`cluster nodes`命令可以查看节点信息，发现符合原来3主3从的预期。

![image-20200926115813557](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200926115813557.png)

## Springboot使用Redis集群

pom文件导入依赖

```xml
        <!--redis依赖配置-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

配置文件

```yaml
spring:
  redis:
    password: # Redis服务器连接密码（默认为空）
    timeout: 3000ms # 连接超时时间
    lettuce:
      pool:
        max-active: 8 # 连接池最大连接数
        max-idle: 8 # 连接池最大空闲连接数
        min-idle: 0 # 连接池最小空闲连接数
        max-wait: -1ms # 连接池最大阻塞等待时间，负值表示没有限制
    cluster:
      nodes:
        - 公网ip:6391
        - 公网ip:6392
        - 公网ip:6393
        - 公网ip:6394
        - 公网ip:6395
        - 公网ip:6396

```

启动即可连接Redis集群