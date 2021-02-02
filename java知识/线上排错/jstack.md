# Jstack 及Arthas 排查CPU 100%

## Jstack

1. 使用TOP 查看CPU占用最高的进程 假如为 1379
2. top -H -p 1379查看占用线程  假如线程高cpu为 1449
3. 将线程ID转换为16进制串输出（用于抓取线程ID堆栈信息 ) 使用 printf "%x\n" 1449 得到 5a9
4. 使用jstack命令抓取堆栈信息（利用16进制）

jatack 进程id | grep 线程16ID进制 -A 行数

jatack 1379 | grep 5a9 -A90



jvisualvm

## Arthas

启动Arthas jar包

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

选择应用java进程：

```bash
* [1]: 35542
  [2]: 71560 arthas-demo.jar
```

Demo进程是第2个，则输入2，再输入`回车/enter`。Arthas会attach到目标进程上，并输出日志：

```bash
[INFO] Try to attach process 71560
[INFO] Attach process 71560 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'
 
wiki: https://arthas.aliyun.com/doc
version: 3.0.5.20181127201536
pid: 71560
time: 2018-11-28 19:16:24
 
$
```

几个常用的命令

```bash
thread -n 3  #查出线程cpu占用最高的三个线程
thread -b    #查看是否有阻塞的线程
dashboard    #查看当前进程信息
```

