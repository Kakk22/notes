# Nginx

## nginx 常用命令



**查看nginx版本号**

```bash
[root@izwz97cxmmtvxp35dd9rxzz sbin]# ./nginx -v
nginx version: nginx/1.19.1
```

**启动nginx**

```bash
[root@izwz97cxmmtvxp35dd9rxzz sbin]# ./nginx
[root@izwz97cxmmtvxp35dd9rxzz sbin]# ps -ef | grep nginx ##查找nginx进程
root     24557     1  0 22:14 ?        00:00:00 nginx: master process ./nginx
nobody   24558 24557  0 22:14 ?        00:00:00 nginx: worker process
root     24560 24524  0 22:15 pts/1    00:00:00 grep --color=auto nginx
```

**关闭nginx**

```bash
[root@izwz97cxmmtvxp35dd9rxzz sbin]# ./nginx -s stop
```

**重新加载nginx**

```bash
[root@izwz97cxmmtvxp35dd9rxzz sbin]# ./nginx -s reload
```

## nginx配置文件

**第一部分 全局块**

```nginx
worker_processes  1; ##值越大 可以支持并发处理量越多
```

**第二部分 events块**

```nginx
events {
    worker_connections  1024; ##最大连接数
}
```

**第三部分 http全局块**

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #

```

### 反向代理

将所有请求反向代理到8000端口服务

```nginx
      location / {
           proxy_pass http:///localhost:8000/;
        }
```

业务场景： 把请求过来的`url` 带`api`  反向方向代理到8000端口服务

```nginx
        location /api/ {
             proxy_pass http://localhost:8000/;
        }
```

### 配置https证书

先将https证书放到`nginx conf`文件夹中

配置文件如下

```nginx
    server {
        listen       443 ssl;
        # 自己的域名
        server_name  blogmark.top www.blogmark.top;
        #https申请的证书
        ssl_certificate      1_blogmark.top_bundle.crt;
        ssl_certificate_key  2_blogmark.top.key;
        ssl_session_timeout 5m;
        #请按照以下协议配置
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;
        #反向代理
        location / {
           root html;
           index index.html index.htm;
           client_max_body_size 10M;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "Upgrade";
           proxy_set_header HOST $host;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_pass http://localhost:8090/;
        }
    }

```

