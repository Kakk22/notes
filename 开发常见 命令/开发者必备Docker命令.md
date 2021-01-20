# [开发者必备Docker命令](http://www.macrozheng.com/#/reference/docker?id=开发者必备docker命令)

> 本文主要讲解Docker环境的安装以及Docker常用命令的使用，掌握这些对Docker环境下应用的部署具有很大帮助。

## [Docker 简介](http://www.macrozheng.com/#/reference/docker?id=docker-简介)

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的 Linux或Windows机器上。使用Docker可以更方便低打包、测试以及部署应用程序。

## [Docker 环境安装](http://www.macrozheng.com/#/reference/docker?id=docker-环境安装)

- 安装yum-utils：

  ```bash
  yum install -y yum-utils device-mapper-persistent-data lvm2Copy to clipboardErrorCopied
  ```

- 为yum源添加docker仓库位置：

  ```bash
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repoCopy to clipboardErrorCopied
  ```

- 安装docker:

  ```bash
  yum install docker-ceCopy to clipboardErrorCopied
  ```

- 启动docker:

  ```bash
  systemctl start dockerCopy to clipboardErrorCopied
  ```

## [Docker 镜像常用命令](http://www.macrozheng.com/#/reference/docker?id=docker-镜像常用命令)

### [搜索镜像](http://www.macrozheng.com/#/reference/docker?id=搜索镜像)

```bash
docker search javaCopy to clipboardErrorCopied
```

![img](http://www.macrozheng.com/images/refer_screen_51.png)

### [下载镜像](http://www.macrozheng.com/#/reference/docker?id=下载镜像)

```bash
docker pull java:8Copy to clipboardErrorCopied
```

### [如何查找镜像支持的版本](http://www.macrozheng.com/#/reference/docker?id=如何查找镜像支持的版本)

> 由于docker search命令只能查找出是否有该镜像，不能找到该镜像支持的版本，所以我们需要通过docker hub来搜索支持的版本。

- 进入docker hub的官网，地址：[https://hub.docker.com](https://hub.docker.com/)

- 然后搜索需要的镜像： ![img](http://www.macrozheng.com/images/refer_screen_52.png)

- 查看镜像支持的版本： ![img](http://www.macrozheng.com/images/refer_screen_53.png)

- 进行镜像的下载操作：

  ```bash
  docker pull nginx:1.17.0Copy to clipboardErrorCopied
  ```

### [列出镜像](http://www.macrozheng.com/#/reference/docker?id=列出镜像)

```bash
docker imagesCopy to clipboardErrorCopied
```

![img](http://www.macrozheng.com/images/refer_screen_54.png)

### [删除镜像](http://www.macrozheng.com/#/reference/docker?id=删除镜像)

- 指定名称删除镜像

  ```bash
  docker rmi java:8Copy to clipboardErrorCopied
  ```

- 指定名称删除镜像（强制）

  ```bash
  docker rmi -f java:8Copy to clipboardErrorCopied
  ```

- 删除所有没有引用的镜像

  ```bash
  docker rmi `docker images | grep none | awk '{print $3}'`Copy to clipboardErrorCopied
  ```

- 强制删除所有镜像

  ```bash
  docker rmi -f $(docker images)Copy to clipboardErrorCopied
  ```

## [Docker 容器常用命令](http://www.macrozheng.com/#/reference/docker?id=docker-容器常用命令)

### [新建并启动容器](http://www.macrozheng.com/#/reference/docker?id=新建并启动容器)

```bash
docker run -p 80:80 --name nginx -d nginx:1.17.0Copy to clipboardErrorCopied
```

- -d选项：表示后台运行
- --name选项：指定运行后容器的名字为nginx,之后可以通过名字来操作容器
- -p选项：指定端口映射，格式为：hostPort:containerPort

### [列出容器](http://www.macrozheng.com/#/reference/docker?id=列出容器)

- 列出运行中的容器：

  ```bash
  docker psCopy to clipboardErrorCopied
  ```

  

- 列出所有容器

  ```bash
  docker ps -aCopy to clipboardErrorCopied
  ```

  

  ### [停止容器](http://www.macrozheng.com/#/reference/docker?id=停止容器)

  ```bash
  # $ContainerName及$ContainerId可以用docker ps命令查询出来
  docker stop $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

  比如：

  ```bash
  docker stop nginx
  #或者
  docker stop c5f5d5125587Copy to clipboardErrorCopied
  ```

  ### [强制停止容器](http://www.macrozheng.com/#/reference/docker?id=强制停止容器)

  ```bash
  docker kill $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

  ### [启动已停止的容器](http://www.macrozheng.com/#/reference/docker?id=启动已停止的容器)

  ```bash
  docker start $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

  ### [进入容器](http://www.macrozheng.com/#/reference/docker?id=进入容器)

- 先查询出容器的pid：

  ```bash
  docker inspect --format "{{.State.Pid}}" $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

- 根据容器的pid进入容器：

  ```bash
  nsenter --target "$pid" --mount --uts --ipc --net --pidCopy to clipboardErrorCopied
  ```

  

  ### [删除容器](http://www.macrozheng.com/#/reference/docker?id=删除容器)

- 删除指定容器：

  ```bash
  docker rm $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

- 按名称删除容器

  ```bash
  docker rm `docker ps -a | grep mall-* | awk '{print $1}'`Copy to clipboardErrorCopied
  ```

- 强制删除所有容器；

  ```bash
  docker rm -f $(docker ps -a -q)Copy to clipboardErrorCopied
  ```

  ### [查看容器的日志](http://www.macrozheng.com/#/reference/docker?id=查看容器的日志)

- 查看当前全部日志

  ```bash
  docker logs $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

- 动态查看日志

  ```bash
  docker logs $ContainerName(或者$ContainerId) -fCopy to clipboardErrorCopied
  ```

  

  ### [查看容器的IP地址](http://www.macrozheng.com/#/reference/docker?id=查看容器的ip地址)

  ```bash
  docker inspect --format '{{ .NetworkSettings.IPAddress }}' $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

  

  ### [修改容器的启动方式](http://www.macrozheng.com/#/reference/docker?id=修改容器的启动方式)

  ```bash
  docker container update --restart=always $ContainerNameCopy to clipboardErrorCopied
  ```

  ### [同步宿主机时间到容器](http://www.macrozheng.com/#/reference/docker?id=同步宿主机时间到容器)

  ```bash
  docker cp /etc/localtime $ContainerName(或者$ContainerId):/etc/Copy to clipboardErrorCopied
  ```

  ### [指定容器时区](http://www.macrozheng.com/#/reference/docker?id=指定容器时区)

  \```bash docker run -p 80:80 --name nginx \

- e TZ="Asia/Shanghai" \

- d nginx:1.17.0 ```

  ### [在宿主机查看docker使用cpu、内存、网络、io情况](http://www.macrozheng.com/#/reference/docker?id=在宿主机查看docker使用cpu、内存、网络、io情况)

- 查看指定容器情况：

  ```bash
  docker stats $ContainerName(或者$ContainerId)Copy to clipboardErrorCopied
  ```

  

- 查看所有容器情况：

  ```bash
  docker stats -aCopy to clipboardErrorCopied
  ```

  

  ### [查看Docker磁盘使用情况](http://www.macrozheng.com/#/reference/docker?id=查看docker磁盘使用情况)

  ```bash
  docker system dfCopy to clipboardErrorCopied
  ```

  

  ### [进入Docker容器内部的bash](http://www.macrozheng.com/#/reference/docker?id=进入docker容器内部的bash)

  ```bash
  docker exec -it $ContainerName /bin/bashCopy to clipboardErrorCopied
  ```

  

  ### [Docker创建外部网络](http://www.macrozheng.com/#/reference/docker?id=docker创建外部网络)

  ```bash
  docker network create -d bridge my-bridge-networkCopy to clipboardErrorCopied
  ```

## [修改Docker镜像的存放位置](http://www.macrozheng.com/#/reference/docker?id=修改docker镜像的存放位置)

- 查看Docker镜像的存放位置：

  ```bash
  docker info | grep "Docker Root Dir"Copy to clipboardErrorCopied
  ```

  

- 关闭Docker服务：

  ```bash
  systemctl stop dockerCopy to clipboardErrorCopied
  ```

- 移动目录到目标路径：

  ```bash
  mv /var/lib/docker /mydata/dockerCopy to clipboardErrorCopied
  ```

- 建立软连接：

  ```bash
  ln -s /mydata/docker /var/lib/dockerCopy to clipboardErrorCopied
  ```

  

   

  