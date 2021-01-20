# Docker Compose 编排

docker Compose是用于定义和运行多个docker容器应用的工具。编写docker-compose.yaml 文件 一键部署多个服务。



## 安装

**下载Docker Compose**

```shell
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

**修改文件的权限**

```shell
chmod +x /usr/local/bin/docker-compose
```

**查看是否安装成功**

```shell
docker-compose --version

[root@localhost ~]# docker-compose --version
docker-compose version 1.23.2, build 1110ad01
```

## 使用Dokcer Compose的步骤

- 使用Dockerfile定义应用程序环境，一般需要修改初始镜像行为时才需要使用；
- 使用docker-compose.yml定义需要部署的应用程序服务，以便执行脚本一次性部署；
- 使用docker-compose up命令将所有应用服务一次性部署起来。

## docker-compose.yml 常用命令

### -images

指定运行镜像名称

```yaml
##${PROJECT}和{TAG} 为同目录下.env 配置   
  image: ${PROJECT}/user-service:${TAG}
```

### -container_name

```yaml
 #配置容器的名称
 container_name: user-service
```

### -ports

```yaml
#指定宿主机和容器的端口映射
ports:
	-8010:8010
```

### -volumes

```yaml
#将外部文件挂载到容器中
volumes:
	-/mydata/mysql/log:/var/log/mysql
	-/mydata/mysql/conf:/etc/mysql
```

### -links

```yaml
#连接其他容器的服务 如redis，mysql
links:
	-db:database
```

-**depends_on**

```yaml
#当前服务下依赖于哪些服务 会去先加载依赖的服务
    depends_on:
    	-payment-service
```

-**restart**

```yaml
    #运行compose 重启服务
    restart: always
```

### -build

服务除了指定基础的镜像，还可以基于一份Dockerfile,再使用up时执行构建dockerfile任务

```yaml
    build:
      context: ./business/user-service ##指定项目路径 可以用相对路径
      dockerfile: Dockerfile            ##指定项目路径下的dockerfile文件
```

### -environment

```yaml
#配置环境变量
    environment:
      - PROFILE=${PROFILE} #读取配置文件给PEOFILE赋值，在具体项目下的dockerfile文件能使用该环境变量
#如下
dockerfile：
CMD ["--spring.profiles.active=${PROFILE}"]
```

结合起来使用

```yaml
version: '3'
services:

  blade-gateway:
    image: ${PROJECT}/blade-gateway:${TAG}
    build:
      context: ./blade-framework/blade-gateway
      dockerfile: Dockerfile
    container_name: blade-gateway
    privileged: true
    restart: always
    environment:
      - PROFILE=${PROFILE}
    depends_on:
      - user-service
    ports:
      - 8001:8001
      
  user-service:
    image: ${PROJECT}/user-service:${TAG}
    build:
      context: ./business/user-service
      dockerfile: Dockerfile
    container_name: user-service
    privileged: true
    restart: always
    environment:
      - PROFILE=${PROFILE}
    ports:
      - 8010:8010
```



## Docker compose 常用命令

### 构建、创建、启动相关容器

```bash
# -d表示在后台运行
docker-compose up -d
```

### 指定文件启动

```bash
docker-compose -f docker-compose.yml up -d
```

### 停止所有相关容器

```bash
docker-compose stop
```

### 列出所有容器信息

```bash
docker-compose ps
```

## Docker compose Idea部署微服务

IDEA安装docker插件，本地安装Docker Quickstart Terminal并启动服务



IDEA新建一个docker服务 指定docker-compose.yaml 文件

 

maven打包整个项目后，运行docker-compose服务。