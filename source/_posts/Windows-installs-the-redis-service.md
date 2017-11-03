---
title: Windows安装Redis服务
date: 2016-11-19 08:52:32
toc: false
tags: [Redis]
categories: Programming Notes

---

### Windows安装Redis
首先**Redis**官方并不支持**Windows**，而**Windows**版的**Redis**是由[**MSOpenTech**](https://github.com/MSOpenTech)提供的支持，所以首先去下载一个[发布版](https://github.com/MSOpenTech/redis/releases)，我选择的是[**Redis-x64-3.2.100.zip**](https://github.com/MSOpenTech/redis/releases/download/win-3.2.100/Redis-x64-3.2.100.zip)压缩包，解压缩得到如下文件列表
```html
EventLog.dll
Redis on Windows Release Notes.docx
Redis on Windows.docx
redis.windows.conf
redis.windows-service.conf
redis-benchmark.exe
redis-benchmark.pdb
redis-check-aof.exe
redis-check-aof.pdb
redis-cli.exe
redis-cli.pdb
redis-server.exe
redis-server.pdb
Windows Service Documentation.docx
```
有了这些文件之后，就可以开始安装服务了。
```
D:\Redis-x64-3.2.100>redis-server.exe  redis.windows.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.100 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 33824
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[33824] 10 Nov 15:32:54.884 # Server started, Redis version 3.2.100
[33824] 10 Nov 15:32:54.885 * The server is now ready to accept connections on p
ort 6379
```
在启动成功之后，可以使用自带的客户端工具进行测试。双击打开**“redis-cli.exe”**，如果成功连接上之后就可以进行**set**及**get**等操作了。如果需要帮助，可以输入**help**查看。

```
127.0.0.1:6379> set "key" "testValue"
OK
127.0.0.1:6379> get "key"
"testValue"
127.0.0.1:6379> help
redis-cli 3.2.100
To get help about Redis commands type:
      "help @<group>" to get a list of commands in <group>
      "help <command>" for help on <command>
      "help <tab>" to get a list of possible help topics
      "quit" to exit

To set redis-cli perferences:
      ":set hints" enable online hints
      ":set nohints" disable online hints
Set your preferences in ~/.redisclirc
```

再输入**help**空格，然后敲**tab**键，可以像命令提示符一样显示候选项。通常不需要每次都使用**cmd**界面去操作，所以附上一些快速处理的脚本，注意在安装服务之前先关闭防火墙和杀毒软件，否则安装不成功。

### 快速操作脚本

**service-install.bat**
>redis-server --service-install redis.windows.conf --loglevel verbose
pause

安装成功之后输出结果：
>D:\Redis-x64-3.2.100>redis-server --service-install redis.windows.conf --logleve
l verbose
[13868] 26 Dec 09:39:58.364 # Granting read/write access to 'NT AUTHORITY\Networ
kService' on: "D:\Redis-x64-3.2.100" "D:\Redis-x64-3.2.100\"
[13868] 26 Dec 09:39:58.365 # Redis successfully installed as a service.
D:\Redis-x64-3.2.100>pause
请按任意键继续. . .

**uninstall-service.bat**
>redis-server --service-uninstall
pause

卸载成功之后输出结果：
>D:\Redis-x64-3.2.100>redis-server --service-uninstall
[11368] 26 Dec 09:45:47.460 # Redis service successfully uninstalled.
D:\Redis-x64-3.2.100>pause
请按任意键继续. . .

注意在卸载服务之前，需要先去**控制面板|管理工具|服务|停止Redis服务**，然后才能卸载成功。

**starting-service.bat**
>redis-server --service-start
pause

服务启动成功之后输出：
>D:\Redis-x64-3.2.100>redis-server --service-start
[20120] 26 Dec 10:12:30.946 # Redis service successfully started.
D:\Redis-x64-3.2.100>pause
请按任意键继续. . .

**stopping-service.bat**
>redis-server --service-stop
pause

服务停止成功之后输出：
>D:\Redis-x64-3.2.100>redis-server --service-stop
[19612] 26 Dec 10:19:27.974 # Redis service successfully stopped.
D:\Redis-x64-3.2.100>pause
请按任意键继续. . .

命名服务，一个可选的参数可以用来指定安装的服务的名称，这个参数可以跟在service-install、service-start、service-stop或service-uninstall命令的后面。比如以下命令会安装和启动三个独立的Redis实例
>redis-server --service-install --service-name redisService1 --port 10001
redis-server --service-start --service-name redisService1
redis-server --service-install --service-name redisService2 --port 10002
redis-server --service-start --service-name redisService2
redis-server --service-install --service-name redisService3 --port 10003
redis-server --service-start --service-name redisService3

### Windows配置主从Redis
先将Redis目录复制一份，命名为**Redis-x64-3.2-Slave**，并将原Redis目录命名为**Redis-x64-3.2-Master**，注释掉master目录redis.windows.conf文件的集群配置，或者将其设置为**no**
>**\# cluster-enabled yes**
**\# 或者设置为no**
**cluster-enabled no**

修改slave目录redis.windows.conf的端口号为6380，同样注释掉集群配置或设置为**no**，在slaveof后面添加master机器的IP和端口号
>**port 6380**
**slaveof 127.0.0.1 6379**
**\# cluster-enabled yes**
**\# 或者设置为no**
**cluster-enabled no**

然后先启动Master，之后再启动Slave机器，输出如下

```
D:\Redis-x64-3.2-Master>redis-server redis.windows.conf
[20048] 26 Dec 13:04:49.064 * Node configuration loaded, I'm f1c7db8c98b01e9fbd5
089b1231ca0e2d23a1766
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.100 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in cluster mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 20048
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[20048] 26 Dec 13:04:49.066 # Server started, Redis version 3.2.100
[20048] 26 Dec 13:04:49.067 * DB loaded from disk: 0.000 seconds
[20048] 26 Dec 13:04:49.067 * The server is now ready to accept connections on p
ort 6379
[20048] 26 Dec 13:05:14.723 * Slave 127.0.0.1:6380 asks for synchronization
[20048] 26 Dec 13:05:14.723 * Full resync requested by slave 127.0.0.1:6380
[20048] 26 Dec 13:05:14.724 * Starting BGSAVE for SYNC with target: disk
[20048] 26 Dec 13:05:14.726 * Background saving started by pid 26568
[20048] 26 Dec 13:05:14.867 # fork operation complete
[20048] 26 Dec 13:05:14.867 * Background saving terminated with success
[20048] 26 Dec 13:05:14.869 * Synchronization with slave 127.0.0.1:6380 succeede
d
```

```
D:\Redis-x64-3.2-Slave>redis-server redis.windows.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.100 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6380
 |    `-._   `._    /     _.-'    |     PID: 21036
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[21036] 26 Dec 13:05:14.720 # Server started, Redis version 3.2.100
[21036] 26 Dec 13:05:14.721 * DB loaded from disk: 0.001 seconds
[21036] 26 Dec 13:05:14.721 * The server is now ready to accept connections on p
ort 6380
[21036] 26 Dec 13:05:14.721 * Connecting to MASTER 127.0.0.1:6379
[21036] 26 Dec 13:05:14.722 * MASTER <-> SLAVE sync started
[21036] 26 Dec 13:05:14.722 * Non blocking connect for SYNC fired the event.
[21036] 26 Dec 13:05:14.722 * Master replied to PING, replication can continue..
.
[21036] 26 Dec 13:05:14.722 * Partial resynchronization not possible (no cached
master)
[21036] 26 Dec 13:05:14.726 * Full resync from master: a326a4da325133537c655c16b
3a6c2546620f816:1
[21036] 26 Dec 13:05:14.868 * MASTER <-> SLAVE sync: receiving 75 bytes from mas
ter
[21036] 26 Dec 13:05:14.871 * MASTER <-> SLAVE sync: Flushing old data
[21036] 26 Dec 13:05:14.871 * MASTER <-> SLAVE sync: Loading DB in memory
[21036] 26 Dec 13:05:14.872 * MASTER <-> SLAVE sync: Finished with success
```
如果在配置中不小心启用了**cluster-enabled yes**的话，会抛**slaveof directive not allowed in cluster mode**和**(error) CLUSTERDOWN Hash slot not served**异常。

### 主从测试
在主机中使用set，添加一个key之后
>D:\Redis-x64-3.2-Master>redis-cli
127.0.0.1:6379> set "mm" "mm in master"
OK

在从机中使用get，得到这个key的value
>D:\Redis-x64-3.2-Slave>redis-cli
127.0.0.1:6379> get "mm"
"mm in master"

反过来，在从机使用set之后也可以同步到master
>D:\Redis-x64-3.2-Slave>redis-cli
127.0.0.1:6379> set "kk" "mm in slave"
OK

在主机get得到value
>D:\Redis-x64-3.2-Master>redis-cli
127.0.0.1:6379> get "kk"
"mm in slave"

注意，redis的主从配置和集群配置方法是不同的，从机一般是只读的，主机用来进行写操作，一个主数据库可以有多个从数据库，而一个从数据库只能有一个主数据库。一般redis的持久化策略是：
>save 900 1    #当有一条Keys数据被改变时，900秒刷新到Disk一次
save 300 10   #当有10条Keys数据被改变时，300秒刷新到Disk一次
save 60 10000 #当有10000条Keys数据被改变时，60秒刷新到Disk一次

Redis的主从复制是建立在内存快照的持久化基础上的，只要有Slave就一定会有内存快照发生。可以很明显的看到，RDB有它的不足，就是一旦数据库出现问题，那么我们的RDB文件中保存的数据并不是全新的。从上次RDB文件生成到Redis停机这段时间的数据全部丢掉了。

AOF(Append-Only File)比RDB方式有更好的持久化性。由于在使用AOF持久化方式时，Redis会将每一个收到的写命令都通过Write函数追加到文件中，类似于MySQL的binlog。当Redis重启是会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。对应的设置参数为：
>appendonly yes #启用AOF持久化方式
appendfilename appendonly.aof #AOF文件的名称，默认为appendonly.aof
appendfsync always #每次收到写命令就立即强制写入磁盘，是最有保证的完全的持久化，但速度也是最慢的，一般不推荐使用。
appendfsync everysec #每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，是受推荐的方式。
appendfsync no #完全依赖OS的写入，一般为30秒左右一次，性能最好但是持久化最没有保证，不被推荐。

AOF的完全持久化方式同时也带来了另一个问题，持久化文件会变得越来越大。比如我们调用INCR test命令100次，文件中就必须保存全部的100条命令，但其实99条都是多余的。因为要恢复数据库的状态其实文件中保存一条SET test 100就够了。为了压缩AOF的持久化文件，Redis提供了bgrewriteaof命令。收到此命令后Redis将使用与快照类似的方式将内存中的数据以命令的方式保存到临时文件中，最后替换原来的文件，以此来实现控制AOF文件的增长。由于是模拟快照的过程，因此在重写AOF文件时并没有读取旧的AOF文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的AOF文件。

>no-appendfsync-on-rewrite yes #在日志重写时，不进行命令追加操作，而只是将其放在缓冲区里，避免与命令的追加造成DISK IO上的冲突。
auto-aof-rewrite-percentage 100 #当前AOF文件大小是上次日志重写得到AOF文件大小的二倍时，自动启动新的日志重写过程。
auto-aof-rewrite-min-size 64mb #当前AOF文件启动新的日志重写过程的最小值，避免刚刚启动Reids时由于文件尺寸较小导致频繁的重写。


**参考文献**
[1] http://blog.chinaunix.net/uid-20682890-id-3603246.html