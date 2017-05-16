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
[20048] 26 Dec 13:0