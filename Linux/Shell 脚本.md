# Shell 脚本

> 编写脚本自动化处理

## 1、文本处理工具

### 1.1、grep工具

> grep 是行过滤工具 根据关键字进行行过滤

#### 语法和选项

**语法：**

```bash
grep [选项] '关键字' 文件名
```

**常用选项**

```bash
OPTIONS:
    -i: 不区分大小写
    -v: 查找不包含指定内容的行,反向选择
    -w: 按单词搜索
    -o: 打印匹配关键字
    -c: 统计匹配到的行数
    -n: 显示行号
    -r: 逐层遍历目录查找
    -A: 显示匹配行及后面多少行    
    -B: 显示匹配行及前面多少行
    -C: 显示匹配行前后多少行
    -l：只列出匹配的文件名
    -L：列出不匹配的文件名
    -e: 使用正则匹配
    -E:使用扩展正则匹配
    ^key:以关键字开头
    key$:以关键字结尾
    ^$:匹配空行
    --color=auto ：可以将找到的关键词部分加上颜色的显示
```

**颜色显示（别名设置）：**

```powershell
临时设置：
# alias grep='grep --color=auto'            //只针对当前终端和当前用户生效

永久设置：
1）全局（针对所有用户生效）
vim /etc/bashrc
alias grep='grep --color=auto'
source /etc/bashrc

2）局部（针对具体的某个用户）
vim ~/.bashrc
alias grep='grep --color=auto'
source ~/.bashrc
```

**举例说明：**

```powershell
# grep -i root passwd                        忽略大小写匹配包含root的行
# grep -w ftp passwd                         精确匹配ftp单词
# grep -w hello passwd                         精确匹配hello单词;自己添加包含hello的行到文件
# grep -wo ftp passwd                         打印匹配到的关键字ftp
# grep -n root passwd                         打印匹配到root关键字的行好
# grep -ni root passwd                         忽略大小写匹配统计包含关键字root的行
# grep -nic root passwd                        忽略大小写匹配统计包含关键字root的行数
# grep -i ^root passwd                         忽略大小写匹配以root开头的行
# grep bash$ passwd                             匹配以bash结尾的行
# grep -n ^$ passwd                             匹配空行并打印行号
# grep ^# /etc/vsftpd/vsftpd.conf        匹配以#号开头的行
# grep -v ^# /etc/vsftpd/vsftpd.conf    匹配不以#号开头的行
# grep -A 5 mail passwd                      匹配包含mail关键字及其后5行
# grep -B 5 mail passwd                      匹配包含mail关键字及其前5行
# grep -C 5 mail passwd                     匹配包含mail关键字及其前后5行
```

### 1.2、bash特性

#### 命令和文件名自动补全

`tab`键

#### 常见的快捷键

```powershell
^c               终止前台运行的程序
^z                  将前台运行的程序挂起到后台
^d               退出 等价exit
^l               清屏 
^a |home      光标移到命令行的最前端
^e |end      光标移到命令行的后端
^u               删除光标前所有字符
^k               删除光标后所有字符
^r                 搜索历史命令
```

#### 常用的通配符（重点）

```powershell
*:    匹配0或多个任意字符
?:    匹配任意单个字符
[list]:    匹配[list]中的任意单个字符,或者一组单个字符   [a-z]
[!list]: 匹配除list中的任意单个字符
{string1,string2,...}：匹配string1,string2或更多字符串

# rm -f file*
# cp *.conf  /dir1
# touch file{1..5}
```

**bash中的引号（重点）**

- 双引号"" :会把引号的内容当成整体来看待，允许通过$符号引用其他变量值
- 单引号'' :会把引号的内容当成整体来看待，禁止引用其他变量值，shell中特殊符号都被视为普通字符
- 反撇号`` :反撇号和$()一样，引号或括号里的命令会优先执行，如果存在嵌套，反撇号不能用

```powershell
[root@localhost ~]# echo "$(date +%F)"
2020-09-18
[root@localhost ~]# echo '$(date +%F)'
$(date +%F)
```

## 2、shell脚本

### 2.1、什么是shell脚本？

- 一句话概括

简单来说就是将需要执行的命令保存到文本中，按照顺序执行。它是解释型的，意味着不需要编译。

- 准确叙述

**若干命令 + 脚本的基本格式 + 脚本特定语法 + 思想= shell脚本**

### 2.2、 什么时候用到脚本?

**重复化**、复杂化的工作，通过把工作的命令写成脚本，以后仅仅需要执行脚本就能完成这些工作。

### 2.3、shell脚本能干啥?

①自动化软件部署 LAMP/LNMP/Tomcat...

②自动化管理 系统初始化脚本、批量更改主机密码、推送公钥...

③自动化分析处理 统计网站访问量

④自动化备份 数据库备份、日志转储...

⑤自动化监控脚本

### 2.4、如何学习shell脚本？

1. 尽可能记忆更多的命令(记忆命令使用功能和场景)
2. 掌握脚本的标准的格式（指定魔法字节、使用标准的执行方式运行脚本）
3. 必须**熟悉掌握**脚本的基本语法（重点)

### 2.5、学习shell脚本的秘诀

多看（看懂）——>模仿（多练）——>多思考（多写）

### 2.6、shell脚本的基本写法

1）**脚本第一行**，魔法字符**#!**指定解释器【必写】

`#!/bin/bash` 表示以下内容使用bash解释器解析

**注意：** 如果直接将解释器路径写死在脚本里，可能在某些系统就会存在找不到解释器的兼容性问题，所以可以使用:**`#!/bin/env 解释器`** **`#!/bin/env bash`**

2）**脚本第二部分**，注释(#号)说明，对脚本的基本信息进行描述【可选】

```powershell
#!/bin/env bash

# 以下内容是对脚本的基本信息的描述
# Name: 名字
# Desc:描述describe
# Path:存放路径
# Usage:用法
# Update:更新时间

#下面就是脚本的具体内容
commands
...
```

3）**脚本第三部分**，脚本要实现的具体代码内容

### 2.7、shell脚本的执行方法

- 标准脚本执行方法（建议）

```powershell
1) 编写人生第一个shell脚本
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# cat first_shell.sh
#!/bin/env bash

# 以下内容是对脚本的基本信息的描述
# Name: first_shell.sh
# Desc: num1
# Path: /shell01/first_shell.sh
# Usage:/shell01/first_shell.sh
# Update:2019-05-05

echo "hello world"
echo "hello world"
echo "hello world"

2) 脚本增加可执行权限
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# chmod +x first_shell.sh

3) 标准方式执行脚本
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# pwd
/shell01
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# /shell01/first_shell.sh
或者
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# ./first_shell.sh

注意：标准执行方式脚本必须要有可执行权限。
```

- 非标准的执行方法（不建议）
- 直接在命令行指定解释器执行

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# bash first_shell.sh
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# sh first_shell.sh
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# bash -x first_shell.sh
+ echo 'hello world'
hello world
+ echo 'hello world'
hello world
+ echo 'hello world'
hello world
----------------
-x:一般用于排错，查看脚本的执行过程
-n:用来查看脚本的语法是否有问题
------------
```

1. 使用`source`命令读取脚本文件,执行文件里的代码

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# source first_shell.sh
hello world
hello world
hello world
```

## 3、变量的定义

### 1. 变量是什么？

一句话概括：变量是用来临时保存数据的，该数据是可以变化的数据。

### 2. 什么时候需要定义变量？

- 如果某个内容需要多次使用，并且在代码中**重复出现**，那么可以用变量代表该内容。这样在修改内容的时候，仅仅需要修改变量的值。
- 在代码运作的过程中，可能会把某些命令的执行结果保存起来，后续代码需要使用这些结果，就可以直接使用这个变量。

### 3.变量如何定义？

**变量名=变量值**

变量名：用来临时保存数据的

变量值：就是临时的可变化的数据

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=hello            定义变量A
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A            调用变量A，要给钱的，不是人民币是美元"$"
hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo ${A}        还可以这样调用，不管你的姿势多优雅，总之要给钱
hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=world            因为是变量所以可以变，移情别恋是常事
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A            不管你是谁，只要调用就要给钱
world
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# unset A            不跟你玩了，取消变量
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A            从此，我单身了，你可以给我介绍任何人
```

### 4. 变量的定义规则

虽然可以给变量（变量名）赋予任何值；但是，对于变量名也是要求的！:unamused:

#### 1.变量名区分大小写

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# a=world
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A
hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $a
world
```

#### 2.变量名不能有特殊符号

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# *A=hello
-bash: *A=hello: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# ?A=hello
-bash: ?A=hello: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# @A=hello
-bash: @A=hello: command not found

特别说明：对于有空格的字符串给变量赋值时，要用引号引起来
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=hello world
-bash: world: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A="hello world"
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A='hello world'
```

#### 3.变量名不能以数字开头

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# 1A=hello
-bash: 1A=hello: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A1=hello
注意：不能以数字开头并不代表变量名中不能包含数字呦。
```

#### 4. 等号两边不能有任何空格

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A =123
-bash: A: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A= 123
-bash: 123: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A = 123
-bash: A: command not found
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=123
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A
123
```

#### 5.变量名尽量做到见名知意

```powershell
NTP_IP=10.1.1.1
DIR=/u01/app1
TMP_FILE=/var/log/1.log
...

说明：一般变量名使用大写（小写也可以），不要同一个脚本中变量全是a,b,c等不容易阅读
```

### 5. 变量的定义方式有哪些？

#### 1. 基本方式

> 直接赋值给一个变量

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=1234567
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A
1234567
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo ${A:2:4}        表示从A变量中第3个字符开始截取，截取4个字符
3456

说明：
$变量名 和 ${变量名}的异同
相同点：都可以调用变量
不同点：${变量名}可以只截取变量的一部分，而$变量名不可以
```

#### 2.命令执行结果赋值给变量

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# B=`date +%F`
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $B
2019-04-16
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# C=$(uname -r)
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $C
2.6.32-696.el6.x86_64
```

#### 3. 交互式定义变量(read)

**目的：**让用户自己给变量赋值，比较灵活。

**语法：**`read [选项] 变量名`

**常见选项：**

| 选项 | 释义                                                       |
| ---- | ---------------------------------------------------------- |
| -p   | 定义提示用户的信息                                         |
| -n   | 定义字符数（限制变量值的长度）                             |
| -s   | 不显示（不显示用户输入的内容）                             |
| -t   | 定义超时时间，默认单位为秒（限制用户输入变量值的超时时间） |

**举例说明：**

```powershell
用法1：用户自己定义变量值
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# read name
harry
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $name
harry
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# read -p "Input your name:" name
Input your name:tom
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $name
tom
```

用法2：变量值来自文件

```
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# cat 1.txt 
10.1.1.1 255.255.255.0

[root@izwz97cxmmtvxp35dd9rxzz shellTest]# read ip mask < 1.txt 
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $ip
10.1.1.1
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $mask
255.255.255.0
```

#### 4.定义有类型的变量(declare)

**目的：** 给变量做一些限制，固定变量的类型，比如：整型、只读

**用法：**`declare 选项 变量名=变量值`

**常用选项：**

| 选项 | 释义                       | 举例                                         |
| ---- | -------------------------- | -------------------------------------------- |
| -i   | 将变量看成整数             | declare -i A=123                             |
| -r   | 定义只读变量               | declare -r B=hello                           |
| -a   | 定义普通数组；查看普通数组 |                                              |
| -A   | 定义关联数组；查看关联数组 |                                              |
| -x   | 将变量通过环境导出         | declare -x AAA=123456 等于 export AAA=123456 |

**举例说明：**

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# declare -i A=123
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A
123
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# A=hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $A
0

[root@izwz97cxmmtvxp35dd9rxzz shellTest]# declare -r B=hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $B
hello
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# B=world
-bash: B: readonly variable
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# unset B
-bash: unset: B: cannot unset: readonly variable
```

### 6. 变量的分类

#### 1.本地变量

- **本地变量**：当前用户自定义的变量。当前进程中有效，其他进程及当前进程的子进程无效。

#### 2.环境变量

- 环境变量

  ：当前进程有效，并且能够被

  子进程

  调用。

  - `env`查看当前用户的环境变量
  - `set`查询当前用户的所有变量(临时变量与环境变量)
  - `export 变量名=变量值` 或者 `变量名=变量值；export 变量名`

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# export A=hello        临时将一个本地变量（临时变量）变成环境变量
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# env|grep ^A
A=hello

永久生效：
vim /etc/profile 或者 ~/.bashrc
export A=hello
或者
A=hello
export A

说明：系统中有一个变量PATH，环境变量
export PATH=/usr/local/mysql/bin:$PATH
```

#### 3. 全局变量

- **全局变量**：全局所有的用户和程序都能调用，且继承，新建的用户也默认能调用.
- **解读相关配置文件**

| 文件名              | 说明                               | 备注                                            |
| ------------------- | ---------------------------------- | ----------------------------------------------- |
| $HOME/.bashrc       | 当前用户的bash信息,用户登录时读取  | 定义别名、umask、函数等                         |
| $HOME/.bash_profile | 当前用户的环境变量，用户登录时读取 |                                                 |
| $HOME/.bash_logout  | 当前用户退出当前shell时最后读取    | 定义用户退出时执行的程序等                      |
| /etc/bashrc         | 全局的bash信息，所有用户都生效     |                                                 |
| /etc/profile        | 全局环境变量信息                   | 系统和所有用户都生效                            |
| $HOME/.bash_history | 用户的历史命令                     | history -w 保存历史记录 history -c 清空历史记录 |

**说明：**以上文件修改后，都需要重新source让其生效或者退出重新登录。

- 用户登录

  系统

  读取

  相关文件的顺序

  1. `/etc/profile`
  2. `$HOME/.bash_profile`
  3. `$HOME/.bashrc`
  4. `/etc/bashrc`
  5. `$HOME/.bash_logout`

#### 4.系统变量

- **系统变量(内置bash中变量)** ： shell本身已经固定好了它的名字和作用.

| 内置变量   | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| $?         | 上一条命令执行后返回的状态；状态值为0表示执行正常，非0表示执行异常或错误 |
| $0         | 当前执行的程序或脚本名                                       |
| $#         | 脚本后面接的参数的个数                                       |
| $*         | 脚本后面所有参数，参数当成一个整体输出，每一个变量参数之间以空格隔开 |
| $@         | 脚本后面所有参数，参数是独立的，也是全部输出                 |
| $1~$9      | 脚本后面的位置参数，$1表示第1个位置参数，依次类推            |
| ${10}~${n} | 扩展位置参数,第10个位置变量必须用{}大括号括起来(2位数字以上扩起来) |
| $$         | 当前所在进程的进程号，如`echo $$`                            |
| $！        | 后台运行的最后一个进程号 (当前终端）                         |
| !$         | 调用最后一条命令历史中的参数                                 |

- 进一步了解位置参数`$1~${n}`

```powershell
#!/bin/bash
#了解shell内置变量中的位置参数含义
echo "\$0 = $0"
echo "\$# = $#"
echo "\$* = $*"
echo "\$@ = $@"
echo "\$1 = $1" 
echo "\$2 = $2" 
echo "\$3 = $3" 
echo "\$11 = ${11}" 
echo "\$12 = ${12}"
```

- 进一步了解$*和$@的区别

`$*`：表示将变量看成一个整体 `$@`：表示变量是独立的

```powershell
#!/bin/bash
for i in "$@"
do
echo $i
done

echo "我是分割线="

for i in "$*"
do
echo $i
done

[root@izwz97cxmmtvxp35dd9rxzz shellTest]# bash 3.sh a b c
a
b
c
我是分割线=
a b c
```

## 4、简单四则运算

算术运算：默认情况下，shell就只能支持简单的整数运算

运算内容：加(+)、减(-)、乘(*)、除(/)、求余数（%）

### 1. 四则运算符号

| 表达式 | 举例                          |
| ------ | ----------------------------- |
| $(( )) | echo $((1+1))                 |
| $[ ]   | echo $[10-5]                  |
| expr   | expr 10 / 5                   |
| let    | n=1;let n+=1 等价于 let n=n+1 |

### 2.了解i++和++i

- 对变量的值的影响

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# i=1
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# let i++
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $i
2
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# j=1
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# let ++j
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $j
2
```

- 对表达式的值的影响

```powershell
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# unset i j
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# i=1;j=1
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# let x=i++         先赋值，再运算
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# let y=++j         先运算，再赋值
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $i
2
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $j
2
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $x
1
[root@izwz97cxmmtvxp35dd9rxzz shellTest]# echo $y
2
```

## 5、条件判断语法结构

### 1. 条件判断语法格式

- 格式1： **test** 条件表达式
- 格式2： **[** 条件表达式 ]
- 格式3： **[[** 条件表达式 ]] 支持正则 =~

**特别说明：**

1）[ 我两边都有空格 ]

2）[[ 我两边都有空格 ]]

3) 更多判断，`man test`查看，很多的参数都用来进行条件判断

### 2. 条件判断相关参数

问：你要判断什么？

答：我要判断文件类型，判断文件新旧，判断字符串是否相等，判断权限等等...

#### 1. 判断文件类型

| 判断参数 | 含义                                         |
| -------- | -------------------------------------------- |
| -e       | 判断文件是否存在（任何类型文件）             |
| -f       | 判断文件是否存在并且是一个普通文件           |
| -d       | 判断文件是否存在并且是一个目录               |
| -L       | 判断文件是否存在并且是一个软连接文件         |
| -b       | 判断文件是否存在并且是一个块设备文件         |
| -S       | 判断文件是否存在并且是一个套接字文件         |
| -c       | 判断文件是否存在并且是一个字符设备文件       |
| -p       | 判断文件是否存在并且是一个命名管道文件       |
| -s       | 判断文件是否存在并且是一个非空文件（有内容） |

**举例说明：**

```powershell
test -e file                    只要文件存在条件为真
[ -d /shell01/dir1 ]             判断目录是否存在，存在条件为真
[ ! -d /shell01/dir1 ]        判断目录是否存在,不存在条件为真
[[ -f /shell01/1.sh ]]        判断文件是否存在，并且是一个普通的文件
```

#### 2.判断文件权限

| 判断参数 | 含义                       |
| -------- | -------------------------- |
| -r       | 当前用户对其是否可读       |
| -w       | 当前用户对其是否可写       |
| -x       | 当前用户对其是否可执行     |
| -u       | 是否有suid，高级权限冒险位 |
| -g       | 是否sgid，高级权限强制位   |
| -k       | 是否有t位，高级权限粘滞位  |

#### 3. 判断文件新旧

说明：这里的新旧指的是文件的修改时间。

| 判断参数        | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| file1 -nt file2 | 比较file1是否比file2新                                       |
| file1 -ot file2 | 比较file1是否比file2旧                                       |
| file1 -ef file2 | 比较是否为同一个文件，或者用于判断硬连接，是否指向同一个inode |

#### 4. 判断整数

| 判断参数 | 含义     |
| -------- | -------- |
| -eq      | 相等     |
| -ne      | 不等     |
| -gt      | 大于     |
| -lt      | 小于     |
| -ge      | 大于等于 |
| -le      | 小于等于 |

#### 5.判断字符串

| 判断参数           | 含义                                        |
| ------------------ | ------------------------------------------- |
| -z                 | 判断是否为空字符串，字符串长度为0则成立     |
| -n                 | 判断是否为非空字符串，字符串长度不为0则成立 |
| string1 = string2  | 判断字符串是否相等                          |
| string1 != string2 | 判断字符串是否相不等                        |

#### 6.多重条件判断

| 判断符号 | 含义   | 举例                                              |        |                        |
| -------- | ------ | ------------------------------------------------- | ------ | ---------------------- |
| -a 和 && | 逻辑与 | [ 1 -eq 1 -a 1 -ne 0 ] [ 1 -eq 1 ] && [ 1 -ne 0 ] |        |                        |
| -o 和 \  | \      |                                                   | 逻辑或 | [ 1 -eq 1 -o 1 -ne 1 ] |

**特别说明：**

&& 前面的表达式为真，才会执行后面的代码

|| 前面的表达式为假，才会执行后面的代码

; 只用于分割命令或表达式

##### ① 举例说明

- 数值比较

```powershell
[root@server ~]# [ $(id -u) -eq 0 ] && echo "the user is admin"
[root@server ~]$ [ $(id -u) -ne 0 ] && echo "the user is not admin"
[root@server ~]$ [ $(id -u) -eq 0 ] && echo "the user is admin" || echo "the user is not admin"

[root@server ~]# uid=`id -u`
[root@server ~]# test $uid -eq 0 && echo this is admin
this is admin
[root@server ~]# [ $(id -u) -ne 0 ]  || echo this is admin
this is admin
[root@server ~]# [ $(id -u) -eq 0 ]  && echo this is admin || echo this is not admin
this is admin
[root@server ~]# su - stu1
[stu1@server ~]$ [ $(id -u) -eq 0 ]  && echo this is admin || echo this is not admin
this is not admin
```

- 类C风格的数值比较

```powershell
注意：在(( ))中，=表示赋值；表示判断
[root@server ~]# ((12));echo $?
[root@server ~]# ((1<2));echo $?
[root@server ~]# ((2>=1));echo $?
[root@server ~]# ((2!=1));echo $?
[root@server ~]# ((`id -u`0));echo $?

[root@server ~]# ((a=123));echo $a
[root@server ~]# unset a
[root@server ~]# ((a123));echo $?
```

- 字符串比较

```powershell
注意：双引号引起来，看作一个整体；= 和  在 [ 字符串 ] 比较中都表示判断
[root@server ~]# a='hello world';b=world
[root@server ~]# [ $a = $b ];echo $?
[root@server ~]# [ "$a" = "$b" ];echo $?
[root@server ~]# [ "$a" != "$b" ];echo $?
[root@server ~]# [ "$a" ! "$b" ];echo $?        错误
[root@server ~]# [ "$a"  "$b" ];echo $?
[root@server ~]# test "$a" != "$b";echo $?


test  表达式
[ 表达式 ]
[[ 表达式 ]]

思考：[ ] 和 [[ ]] 有什么区别？

[root@server ~]# a=
[root@server ~]# test -z $a;echo $?
[root@server ~]# a=hello
[root@server ~]# test -z $a;echo $?
[root@server ~]# test -n $a;echo $?
[root@server ~]# test -n "$a";echo $?

# [ '' = $a ];echo $?
-bash: [: : unary operator expected
2
# [[ '' = $a ]];echo $?
0


[root@server ~]# [ 1 -eq 0 -a 1 -ne 0 ];echo $?
[root@server ~]# [ 1 -eq 0 && 1 -ne 0 ];echo $?
[root@server ~]# [[ 1 -eq 0 && 1 -ne 0 ]];echo $?
```

##### ② 逻辑运算符总结

1. 符号;和&&和||都可以用来分割命令或者表达式

2. 分号（;）完全不考虑前面的语句是否正确执行，都会执行;号后面的内容

3. `&&`符号，需要考虑&&前面的语句的正确性，前面语句正确执行才会执行&&后的内容；反之亦然

4. ###### `||`符号，需要考虑||前面的语句的非正确性，前面语句执

## 6、流程控制语句

**关键词：选择**

### 1. 基本语法结构

#### 1.if结构

**箴言1：只要正确，就要一直向前冲:v:**

**F**:表示false，为假

**T**:表示true，为真

```powershell
if [ condition ];then
        command
        command
fi

if test 条件;then
    命令
fi

if [[ 条件 ]];then
    命令
fi

[ 条件 ] && command
```

![流程判断1](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/%E6%B5%81%E7%A8%8B%E5%88%A4%E6%96%AD1.png)

#### 2. if...else结构

**箴言2：分叉路口，二选一**

```powershell
if [ condition ];then
        command1
    else
        command2
fi

[ 条件 ] && command1 || command2
```

![流程判断2](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/%E6%B5%81%E7%A8%8B%E5%88%A4%E6%96%AD2.png)

**小试牛刀：**

==让用户自己输入==字符串，==如果==用户输入的是hello，请打印world，==否则==打印“请输入hello”

1. `read定义变量`
2. if....else...

```powershell
#!/bin/env bash

read -p '请输入一个字符串:' str
if [ "$str" = 'hello' ];then
    echo 'world'
 else
    echo '请输入hello!'
fi

  1 #!/bin/env bash
  2
  3 read -p "请输入一个字符串:" str
  4 if [ "$str" = "hello" ]
  5 then
  6     echo world
  7 else
  8     echo "请输入hello!"
  9 fi

  echo "该脚本需要传递参数"
  1 if [ $1 = hello ];then
  2         echo "hello"
  3 else
  4         echo "请输入hello"
  5 fi

#!/bin/env bash

A=hello
B=world
C=hello

if [ "$1" = "$A" ];then
        echo "$B"
    else
        echo "$C"
fi


read -p '请输入一个字符串:' str;
 [ "$str" = 'hello' ] && echo 'world' ||  echo '请输入hello!'
```

#### 3.if...elif...else结构

**箴言3：选择很多，能走的只有一条**

```powershell
if [ condition1 ];then
        command1      结束
    elif [ condition2 ];then
        command2       结束
    else
        command3
fi
注释：
如果条件1满足，执行命令1后结束；如果条件1不满足，再看条件2，如果条件2满足执行命令2后结束；如果条件1和条件2都不满足执行命令3结束.
```

![流程判断3](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/%E6%B5%81%E7%A8%8B%E5%88%A4%E6%96%AD3.png)

#### 4.层层嵌套结构

**箴言4：多次判断，带你走出人生迷雾。**

```powershell
if [ condition1 ];then
        command1        
        if [ condition2 ];then
            command2
        fi
 else
        if [ condition3 ];then
            command3
        elif [ condition4 ];then
            command4
        else
            command5
        fi
fi
注释：
如果条件1满足，执行命令1；如果条件2也满足执行命令2，如果不满足就只执行命令1结束；
如果条件1不满足，不看条件2；直接看条件3，如果条件3满足执行命令3；如果不满足则看条件4，如果条件4满足执行命令4；否则执行命令5
```

![流程判断4](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/%E6%B5%81%E7%A8%8B%E5%88%A4%E6%96%AD4.png)

### 2. 应用案例

#### 1.判断两台主机是否ping通

**需求：**判断当前主机是否和远程主机是否ping通

##### ① 思路

1. 使用哪个命令实现 `ping -c次数`
2. 根据命令的==执行结果状态==来判断是否通`$?`
3. 根据逻辑和语法结构来编写脚本(条件判断或者流程控制)

##### ② 落地实现

```powershell
#!/bin/env bash
# 该脚本用于判断当前主机是否和远程指定主机互通

# 交互式定义变量，让用户自己决定ping哪个主机
read -p "请输入你要ping的主机的IP:" ip

# 使用ping程序判断主机是否互通
ping -c1 $ip &>/dev/null

if [ $? -eq 0 ];then
    echo "当前主机和远程主机$ip是互通的"
 else
     echo "当前主机和远程主机$ip不通的"
fi

逻辑运算符
test $? -eq 0 &&  echo "当前主机和远程主机$ip是互通的" || echo "当前主机和远程主机$ip不通的"
```

#### 2. 判断一个进程是否存在

**需求：**判断web服务器中httpd进程是否存在

##### ① 思路

1. 查看进程的相关命令 ps pgrep
2. 根据命令的返回状态值来判断进程是否存在
3. 根据逻辑用脚本语言实现

##### ② 落地实现

```powershell
#!/bin/env bash
# 判断一个程序(httpd)的进程是否存在
pgrep httpd &>/dev/null
if [ $? -ne 0 ];then
    echo "当前httpd进程不存在"
else
    echo "当前httpd进程存在"
fi

或者
test $? -eq 0 && echo "当前httpd进程存在" || echo "当前httpd进程不存在"
```

##### ③ 补充命令

```powershell
pgrep命令：以名称为依据从运行进程队列中查找进程，并显示查找到的进程id
选项
-o：仅显示找到的最小（起始）进程号;
-n：仅显示找到的最大（结束）进程号；
-l：显示进程名称；
-P：指定父进程号；pgrep -p 4764  查看父进程下的子进程id
-g：指定进程组；
-t：指定开启进程的终端；
-u：指定进程的有效用户ID。
```

#### 3. 判断一个服务是否正常

**需求：**判断门户网站是否能够正常访问

##### ① 思路

1. 可以判断进程是否存在，用/etc/init.d/httpd status判断状态等方法
2. 最好的方法是直接去访问一下，通过访问成功和失败的返回值来判断
   - Linux环境，wget curl elinks -dump

##### ② 落地实现

```powershell
#!/bin/env bash
# 判断门户网站是否能够正常提供服务

#定义变量
web_server=www.itcast.cn
#访问网站
wget -P /shell/ $web_server &>/dev/null
[ $? -eq 0 ] && echo "当前网站服务是ok" && rm -f /shell/index.* || echo "当前网站服务不ok，请立刻处理"
```

### 3. 练习

#### 1.判断用户是否存在

**需求1：**输入一个用户，用脚本判断该用户是否存在

```powershell
 #!/bin/env bash
  2 read -p "请输入一个用户名：" user_name
  3 id $user_name &>/dev/null
  4 if [ $? -eq 0 ];then
  6     echo "该用户存在！"
  7 else
  8     echo "用户不存在！"
  9 fi


#!/bin/bash
# 判断 用户（id） 是否存在
read -p "输入壹个用户：" id
id $id &> /dev/null
if [ $? -eq 0 ];then
        echo "该用户存在"
else
        echo "该用户不存在"
fi

#!/bin/env bash
read -p "请输入你要查询的用户名:" username
grep -w $username /etc/passwd &>/dev/null
if [ $? -eq 0 ]
then
    echo "该用户已存在"
else
    echo "该用户不存在"
fi

#!/bin/bash
read -p "请输入你要检查的用户名：" name
id $name &>/dev/null
if [ $? -eq 0 ]
then
echo 用户"$name"已经存在
else
echo 用户"$name"不存在
fi

#!/bin/env bash
#判断用户是否存在
read -p "请写出用户名" id
id $id
if [ $? -eq 0 ];then
        echo "用户存在"
else
        echo "用户不存在"
fi

#!/bin/env bash
read -p '请输入用户名:' username
id $username &>/dev/null
[ $? -eq 0 ] && echo '用户存在' || echo '不存在'
```

