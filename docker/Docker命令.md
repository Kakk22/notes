# Docker命令

## Docker环境安装

- 安装yum-utils：

  ```bash
  yum install -y yum-utils device-mapper-persistent-data
  ```

- 为yum源添加docker仓库位置：

  ```bash
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker
  ```

- 安装docker:

  ```bash
  yum install docker
  ```

- 启动docker:

  ```bash
  systemctl start docker
  ```

## Docker 镜像常用命令

### 下载镜像

```bash
docker pull java:8
```

### 列出镜像

```bash
docker images
```

### 删除镜像

- 指定名称删除镜像

  ```bash
  docker rmi java:8
  ```

- 指定名称删除镜像（强制）

  ```bash
  docker rmi -f java:8
  ```

- 删除所有没有引用的镜像

  ```bash
  docker rmi `docker images | grep none | awk '{print $3}'`
  ```

- 强制删除所有镜像

  ```bash
  docker rmi -f $(docker images)
  ```

## Docker 容器常用命令

### 新建并启动容器

```bash
docker run -p 80:80 --name nginx -d nginx:1.17.0
```

- -d选项：表示后台运行
- --name选项：指定运行后容器的名字为nginx,之后可以通过名字来操作容器
- -p选项：指定端口映射，格式为：hostPort:containerPort

### 列出容器

- 列出运行中的容器：

  ```bash
  docker ps
  ```

- 列出所有容器

  ```bash
  docker ps -a
  ```


### 停止容器

```bash
# $ContainerName及$ContainerId可以用docker ps命令查询出来
docker stop $ContainerName(或者$ContainerId)
```

### 强制停止容器

```bash
docker kill $ContainerName(或者$ContainerId)
```

### 启动已停止的容器

```bash
docker start $ContainerName(或者$ContainerId)
```

### 进入容器

```shell
exec -it XXXXX /bin/bash
```

进入redis

```shell
docker exec -it redis redis-cli
```

进入mysql

```shell
docker exec -it mysql /bin/bash	

root@77cc6b447a74:/# mysql -uroot -p   ##输入账号密码
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 60
Server version: 5.7.30 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases; ##查看数据库

```

## 查看容器日志

```shell
docker logs [容器id] -f
```

