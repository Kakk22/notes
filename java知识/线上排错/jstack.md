# Jstack 及Arthas 排查CPU 100%

## Jstack

1. 使用TOP 查看CPU占用最高的进程 假如为 1379
2. top -H -p 1379查看占用线程  假如线程高cpu为 1449
3. 将线程ID转换为16进制串输出（用于抓取线程ID堆栈信息 ) 使用 printf "%x\n" 1449 得到 5a9
4. 使用jstack命令抓取堆栈信息（利用16进制）

jatack 进程id | grep 线程16ID进制 -A 行数

jatack 1379 | grep 5a9 -A90



jvisualvm

