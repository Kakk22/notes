# Dockerfile

## 一、关于dockerfile

简单的示例

```dockerfile
FROM docker.io/huanwei/alpine-oraclejdk8
RUN mkdir -p /mall
WORKDIR /mall
ADD target/*.jar app.jar
ENV LANG C.UTF-8
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom","-Xms128m","-Xmx256m","-jar", "app.jar"]
CMD ["--spring.profiles.active=${PROFILE}"]
```



可以看出dockerfile的结构分为以下几个部分：

1. 基础的镜像信息
2. 镜像的操作命令
3. 容器启动时的执行命令



## 二、dockerfile的常用命令

```dockerfile
FROM #指定基础的镜像，找不到会自动下载
RUN  #想做什么？
ADD  #copy文件，会自动解压
WORKDIR #设置当前的工作目录
VOLUME  #设置卷，挂载主机目录
EXPOSE  #暴露端口
CMD     #指定容器启动后要干什么
```

### 2.1 FROM

指定构建的新镜像来自哪个基础镜像

```dockerfile
FROM docker.io/huanwei/alpine-oraclejdk8 #指定基础的jdk
```

### 2.2 RUN

```dockerfile
RUN mkdir -p /mall #创建目录
RUN yum install httpd #下载资源
```

### 2.3 CMD

```dockerfile
CMD ["--spring.profiles.active=${PROFILE}"] #启动容器时执行的Shell命令
```

### 2.4 EXPOSE

```dockerfile
EXPOSE 80 80 #声明容器运行的服务端口
```

### 2.5 ENV

```dockerfile
ENV LANG C.UTF-8 #设置环境内的环境变量
```

### 2.6 ADD

```dockerfile
ADD target/*.jar app.jar #拷贝文件或目录到镜像中 
```

### 2.7 COPY

```dockerfile
COPY ./start.sh /start.sh #拷贝文件或目录到镜像中，用法同ADD 但是不会自动解压 
```

### 2.8 ENTERYPOINT

```dockerfile
#启动容器执行的Shell命令，同CMD类似，只是由ENTRYPOINT启动的程序不会被docker run命令行指定的参数所覆盖
#而且，这些命令行参数会被当作参数传递给ENTRYPOINT指定指定的程序
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom","-Xms128m","-Xmx256m","-jar", "app.jar"]
```

### 2.9 VOLUME

```dockerfile
#指定容器挂载点到宿主机自动生成的目录或其他容器 ，也可以用docker RUN -V 指定
VOLUME ["/var/lib/mysql"] 
```

### 2.10 WORKDIR

```dockerfile
WORKDIR /mall #为RUN、CMD、ENTRYPOINT以及COPY和AND设置工作目录
```

