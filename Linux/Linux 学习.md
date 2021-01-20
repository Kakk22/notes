# Linux 学习

## -vim常用命令

`  ll`  -查看当前目录文件

` vim -编辑文件 i 进入插入模式，esc 回到正常模式， ：或者/ 进入命令行模式   wq！ 保存退出`   

快捷键：

`yy 复制当前行  5yy复制5行 p粘贴`

`dd 删除当前行  5dd 删除往下5行`

` /+查找内容  表示查找 n查询下一个 N是查询上一个`  

` ：set nu 显示行数 :set nonu 关闭行数`  

`gg 回到顶行  G 回到尾行   20+shift+G 找到20行`

## -开机 重启 用户登录注销

**开机 重启**

```shell
shutdown
  shutdown -h now   //立即关机
  shutdown -h 1     //1分钟后关机
  shutdown -r now   //立即挂机重启
halt   //关机
reboot // 立即关机重启
sync  //把内存数据写入磁盘  关机前执行
```

**用户登录和注销**

`su-"用户名"  切换系统管理员身份`

`logout  退出连接/注销`

## -用户管理

**添加用户**

`useradd  用户名 `

`useradd -d 指定目录  新的用户名 指定目录`

**指定/*修改密码***

`passwd 用户名` 修改密码

**删除目录**

`userdel 用户名`

`uderdel -r 用户名`  家目录也删除

**查询用户信息**

`id 用户名`

```shell
[root@izwz97cxmmtvxp35dd9rxzz ~]# chattr -ia /etc/passwd //移除ia权限
[root@izwz97cxmmtvxp35dd9rxzz ~]# lsattr /etc/passwd    //查看文件权限
```

**切换用户**

`su - 用户名`

**用户组**

`groupadd 组名 ` 添加一个组

`groupdel 组名`  删除一个组

`usermod -g  组名 用户名`  修改用户组

**用户和组的相关文件**

`/etc/passwd 文件`  用户的配置文件，记录用户的各种信息

`/etc/group 文件` 组的配置文件，记录Linux包含的组的信息

`/etc/shadow 文件`  口令配置文件

## -实用指令

### **1、指定运行级别**

```
0： 关机
1： 单用户（找回丢失密码）
2:  多用户状态没有网络
3： 多用户状态有网络
4： 系统未使用保留给用户
5:  图形界面
6： 系统重启
常用的运行级别是3和5
```

![image-20200608160105581](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200608160105581.png)

**如何找回root密码：**

进入 单用户模式，修改root密码。



### **2、帮助命令**

`man [命令或者配置文件]`  获得功能描述

`help [命令或者配置文件]`  获得功能描述

### 3、文件目录类

`psw`  显示当前文件目录 

`ls -a -l  `显示当前目录文件 -a 显示隐藏  -l 列表显示

```shell
cd [参数] 切换到指定目录
cd~ 或者 cd： 回到家目录
cd .. 返回上级目录
cd /root 使用绝对路径到root
cd  ../../root 使用相对路径到root
```

**创建：**

 `mkdir  -p  [目录] ` 创建一个目录, -p 创建多级目录

`touch [文件名]` 创建文件

**删除：**

`rmdir [目录]` 删除目录 不能删除非空

`rm -rf [目录]` 删除非空目录

`rm [选项] [文件名]`  **删除文件或文件夹  选项：-r 递归删除  -f 删除不提醒**

**拷贝：**

`cp  [-r] 文件名  目录 `  **拷贝文件到目标目录  -r递归拷贝 可以拷贝整个文件夹**

`/cp ` 强制覆盖 不提醒

**剪切 重命名：**

`mv  a.txt  hello.txt  `  重命名

`mv hello.txt  /root`  剪切

**查看文件**：

`cat -n  文件 | more   `  查看文件， -n 显示行   | more 分页查看  宫格下一页 shift+/ 上一页

`more  文件    `  查看文件，    宫格/shift+f 下一页   shift+b 上一页

`less  查看的文件`  查看文件，适用于查看大型文件，不是一次全部加载，只加载需要查看的



`>输出重定向 >> 追加`

`ls -l > a.txt 将列表内容写入a.txt   ls -l >> a.txt 将列表内容追加到a.txt`

`cat 文件1 >文件2  将文件1内容覆盖到文件2`

`echo  "内容">> 文件` 将内容追加到文件



`echo   内容` 输出内容到控制台

`head  文件` 显示文件前10行

`tail 文件` **显示文件后10行   -n 5文件 查看后5行  -f 实时追踪文档的所有更新！（经常用）**



`ln -s 原目录或文件  软链接文件`  软链接 类似快捷键

`history ` 查看历史命令

### 4、时间日期类

`date` 当前时间

`date "+%y-%m-%d"`  显示年月日

**设置日期**

`date -s  字符串时间 "2020-6-9 15:21:50"`

显示日历

`cal` 当前日历 

### 5、搜索查找类

`find /目录 -name  文件名称 ` 查找目录下指定名称的文件

`find  /目录  -size  +20M` 查找目录下比20M大的文件

`locate 搜索文件`  locate基于数据库查询，查询速度较快 使用前用`updatedb`

`cat a.txt | grep -ni  yes` 管道符和`grep` 过滤查找， -n 显示行号 -i 忽略大小写  

### 6、解压文件类

`gzip  文件`压缩文件

`gunzip 文件` 解压文件



`zip [选项] 文件名  压缩的目录`压缩文件 选项 -r 递归压缩目录

`zip -r mypack /home/`

`unzip  -d 指定位置  解压文件` 解压文件  -d 指定存放目录

`unzip -d /opt/tmp mypack.zip` 

**tar命令**

打包：

` tar -zcvf a.tar.gz a.txt b.txt` 将`a.txt b.txt `压缩到`a.tar.gz`

`-z 打包同时压缩 -c 产生tar打包文件 -v 显示详细信息 -f 指定压缩后的文件名`

`tar -zcvf myhome.tar.gz  /home/` 压缩home目录下所有文件

解压：

`tar -zxvf a.tar.gz` 解压文件到当前目录

``tar -zxvf a.tar.gz -C   指定目录` 解压到指定目录  目录要存在不然报错  

## -组管理和权限管理

**查看文件的所有者：**

`ls -ahl`

`groupadd police  useradd -g police tom  ls -ahl `创建一个组 把tom放再组里 tom创建的文件都属于这个组

**修改文件所有者：**

`chown tom a.txt`  把`a.txt `文件修改所有者给tom

**创建组：**

`groupadd 组名`

`useradd -g  mostor tom` 把tom放在组`mostor里`

**修改文件所在的组：**

`chgrp 组名 文件`

**改变用户所在的组：**

`usermod -g 组名 用户名`

`usermod -d 目录名 用户名`  改变用户登录的初始目录

**权限基本介绍：**

![image-20200610112051872](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610112051872.png)

![image-20200610112103138](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610112103138.png)

![image-20200610112130528](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610112130528.png)

![image-20200610112221085](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610112221085.png)



**1、修改权限**

`chmod ` 命令可以修改文件或者文件夹的权限

**1.1 通过字母**

u 所有人 g 所有组 o其他人 a所有人

`chmod u=rwx,g=rx,o=x 文件目录名`

`chmod o+w  文件目录名`

`chmod a-x 文件目录名`

**1.2 通过数字**

`r=4 w=2 x =1  rwx=7  `

 `chmod  u=rwx,g=rx,o=x `

`相当于 chmod 751 文件目录名`

**2、修改文件所有者-chown**

`chown newowner file`  改变目录所有者

`chown newowner:newgroup  file` 改变用户所有者和所有组

`-R` 如果是目录 则在其下所有子文件或目录递归生效

![image-20200610141441070](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610141441070.png)

**3、修改所在组-chgrp**

`chgrp newgroup file ` 改变文件所在的组

![image-20200610141556145](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200610141556145.png)

## -**任务调度**

让系统指定时间去做指定的事情

1、基本语法

`crontab [选项]` 

-e 编辑定时任务

-l 查看定时任务

-r 删除当前用户所有的`crontab`任务

![image-20200611110735213](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611110735213.png)

![image-20200611110748374](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611110748374.png)

![image-20200611110802316](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611110802316.png)

**2、应用实例**

![image-20200611140045765](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611140045765.png)

![image-20200611140057677](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611140057677.png)

![image-20200611140113419](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611140113419.png)

![image-20200611140124018](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611140124018.png)

## -磁盘分区，挂载

![image-20200611142313109](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611142313109.png)

window分区

![image-20200611142328622](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611142328622.png)

![image-20200611142406940](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611142406940.png)

![image-20200611142422117](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611142422117.png)

![image-20200611142435335](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611142435335.png)

![image-20200611143847645](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143847645.png)

![image-20200611143856451](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143856451.png)

![image-20200611143907018](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143907018.png)

![image-20200611143916312](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143916312.png)

![image-20200611143933539](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143933539.png)

![image-20200611143948005](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143948005.png)

![image-20200611143957029](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611143957029.png)

![image-20200611145419555](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611145419555.png)

![image-20200611145428645](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611145428645.png)

**磁盘工作常用命令**

1、统计home目录下文件个数

`ls -l /home | grep "^-" | wc -l`

2、统计home目录下文件夹个数

`ls -l /home | grep "^d" | wc -l`

3、统计home目录下文件个数包括文件夹里

`ls -lR /home | grep "^-" | wc -l`  -R 递归

4、统计home目录下文件夹个数包括子文件夹里

`ls -lR /home | grep "^d" | wc -l` 

## -网络配置

![image-20200611151622993](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611151622993.png)

![image-20200611151643445](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611151643445.png)

![image-20200611151652743](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611151652743.png)

## -进程管理

![image-20200611160603809](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160603809.png)

![image-20200611160614765](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160614765.png)

![image-20200611160624511](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160624511.png)

![image-20200611160634488](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160634488.png)

![image-20200611160702949](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160702949.png)

![image-20200611160712223](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611160712223.png)

`ps -ef | grep sshd`

![image-20200611161555888](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611161555888.png)

![image-20200611161608432](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611161608432.png)

![image-20200611161618108](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611161618108.png)

![image-20200611161629611](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200611161629611.png)

## -服务管理

![image-20200612104057408](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104057408.png)

![image-20200612104110558](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104110558.png)

![image-20200612104124588](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104124588.png)

![image-20200612104135151](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104135151.png)

![image-20200612104148967](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104148967.png)

![image-20200612104237309](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104237309.png)

![image-20200612104252740](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104252740.png)

![image-20200612104309894](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612104309894.png)

![image-20200612110210511](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612110210511.png)

![image-20200612110227613](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612110227613.png)

![image-20200612110242234](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612110242234.png)

![image-20200612110258256](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612110258256.png)

![image-20200612110310145](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612110310145.png)

## -yum

![image-20200612151246650](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612151246650.png)

![image-20200612151256263](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612151256263.png)

![image-20200612151305298](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20200612151305298.png)