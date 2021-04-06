# Linux常用命令

## ps

> 查看当前进程状态，经常与grep配合使用

* ps -A   或者ps -e  显示所有进程
* ps -e | grep name 查看符合匹配规则的进程

## netstat

> 查看服务器网络信息

* netstat -t 显示tcp连接信息
* netstat -u 显示udp连接信息
* netstat -a 显示所有信息 
* netstat -l 显示处于Listening状态的连接



## top

> 显示服务器的实时性能，使用uptime命令可以只显示第一行的信息

![image-20210331100447770](C:\Environment\Github\Typora\操作系统\image-20210331100447770.png)

第一行 ：当前时间  系统已运行时间 当前连接用户数 平均负载情况：1分钟、5分钟、15分钟时

第二行：总进程数 运行进程数  休眠进程数  停止进程数 僵尸进程数

第三行：用户占用cpu量4.4% 内核占用量

## lsof

> list opening file(lsof) 查看打开文件

* lsof  [filename] 显示打开指定文件的进程
* lsof -u [username] 显示指定用户进程的文件，如root用户
* lsof -d  [fd] 显示指定文件描述符的进程



