title: "Windows下搭建Solr5.5搜索服务"
date: 2016-03-21 22:29:04
tags: [Solr]
categories: Programming Notes

---

####机器环境
**Windows**：Win7 64 bit
**Java**：java version "1.8.0_45"；Java HotSpot(TM) 64-Bit Server VM
**Solr**：5.5
**Lucene**：5.5
**Tomcat**：8.0.32

Lucene和Solr下载地址：http://lucene.apache.org/
Windows选择下载zip压缩包，Linux选择下载tgz压缩包
Tomcat下载地址：http://tomcat.apache.org/ ，选择Binary Distributions下的Core中的64-bit Windows zip (pgp, md5, sha1)下载之后文件名称是：apache-tomcat-8.0.32-windows-x64

####Windows下启动Tomcat并添加服务
#####配置环境变量
注意**JDK**和**Tomcat**的目录中最好别有中文，首先在系统环境变量中配置**JAVA_HOME**，值是`C:/Program Files/Java/jdk1.8.0_45`，然后在**Path**中添加`;%JAVA_HOME%/bin`。

添加环境变量**CATALINA_HOME**值是`D:/apache-tomcat-8.0.32`，然后点击 **tomcat**的**bin**下的**startup.bat**就应该可以启动**tomcat**了。

#####添加Tomcat到Windows服务
如果想把**tomcat**添加到**Windows**服务中，需要打开**cmd**运行**Tomcat**的**bin**目录下的`service.bat`脚本，安装命令如下，其中**tomcat8**是自己起的服务名称，当然也可以使用其它名称
>D:\apache-tomcat-8.0.32>cd bin
D:\apache-tomcat-8.0.32\bin>service.bat install tomcat8
Installing the service 'tomcat8' ...
Using CATALINA_HOME:    "D:\apache-tomcat-8.0.32"
Using CATALINA_BASE:    "D:\apache-tomcat-8.0.32"
Using JAVA_HOME:        "C:\Program Files\Java\jdk1.8.0_45"
Using JRE_HOME:         "C:\Program Files\Java\jdk1.8.0_45\jre"
Using JVM:              "C:\Program Files\Java\jdk1.8.0_45\jre\bin\server\jvm.dl
l"
The service 'tomcat8' has been installed.

卸载服务命令如下
>D:\apache-tomcat-8.0.32\bin>service.bat remove tomcat8
Removing the service 'tomcat8' ...
Using CATALINA_BASE:    "D:\apache-tomcat-8.0.32"
The service 'tomcat8' has been removed

#####配置Tomcat管理员
在**Tomcat**启动之后，可以在浏览器中输入http://localhost:8080/ 查看是否配置成功，如果出现**Tomcat**主界面，点击**Manager App**，提示权限不足，此时需要在`D:/apache-tomcat-8.0.32/conf/tomcat-users.xml`文件中添加如下内容
```xml
<role rolename="manager-gui"/>
<user username="admin" password="admin" roles="manager-gui"/>
```
重启**Tomcat**，打开浏览器点击**Manager App**会弹出登录框，使用配置的用户名和密码登录即可。

####Tomcat启动Solr
1. 解压从官网下载的**solr-5.5.0.zip**压缩包，解压到**D:/solr-5.5.0**
2. 复制**solr-5.5.0/server/solr-webapp/webapp**到**tomcat**下的**webapps**目录下，改名为**solr**
3. 将**solr-5.5.0/server/lib/ext/**目录下的所有**jar**包复制到**tomcat/webapps/solr/WEB-INF/lib/**下
4. 将**solr-5.5.0/server/solr**目录复制到**D:/apache-tomcat-8.0.32/bin**目录下，这个就是传说中的**solr/home**(存放检索数据)
5. 设置**solr/home**：编辑**D:/apache-tomcat-8.0.32/webapps/solr/WEB-INF/web.xml**文件，以下部分是注释的，打开该注释，并修改**env-entry-value**。**solr**在启动的时候会去这个根目录下加载配置信息。
```xml
<env-entry>
   <env-entry-name>solr/home</env-entry-name>
   <env-entry-value>D:/apache-tomcat-8.0.32/bin/solr</env-entry-value>
   <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```
6. 将**solr-5.5.0/server/resources**下的**log4j.properties**文件复制到**tomcat/webapps/solr/WEB-INF/classes**目录下，如果该目录不存在则新建。
7. 将**solr-5.5.0/dist**目录下的`solr-dataimporthandler-5.5.0.jar`和`solr-dataimporthandler-extras-5.5.0.jar`复制到**tomcat/webapps/solr/WEB-INF/lib/**下，这个是为了以后导入数据库表数据。
配置完成之后，**Solr**环境搭建完毕，重启**Tomcat**，在浏览器中输入http://localhost:8080/solr/index.html#/ 就可以看到**Solr**的控制台了
8. 在**tomcat/solrhome/**目录下创建**core**(自定义)，在其目录下创建**data**文件夹，并将**D:/apache-tomcat-8.0.32/bin/solr/configsets/basic_configs/**目录下的**conf**文件夹复制到**core**下。然后在**solr**控制台点击**Add Core**即可完成创建。
![](http://7xig3q.com1.z0.glb.clouddn.com/Solr_add_core_config.png)

####Jetty启动Solr
**Solr**自带**Jetty**服务器，并且提供了可运行的**jar**包，该**jar**包存在于`D:/solr-5.5.0/server`目录中，但是不能直接双击该**jar**包运行，需要在命令行中进行启动。打开**cmd**命令行，按照网上提示，输入启动命令
>d:\solr-5.5.0\server>java -jar start.jar
WARNING: Nothing to start, exiting ...
Usage: java -jar start.jar [options] [properties] [configs]
       java -jar start.jar --help  # for more information


根据提示信息，该命令并不能启动**jar**包，原因是在新版的**Solr**中并不能以这种方式启动，具体细节可以参见[Solr Quick Start](http://lucene.apache.org/solr/quickstart.html)以及StackOverflow上面的问答[“Nothing to start” when trying to start Apache Solr](http://stackoverflow.com/questions/30983349/nothing-to-start-when-trying-to-start-apache-solr)，根据以上信息，我们采用如下方式启动

>d:\solr-5.5.0\bin>solr start -e cloud -noprompt
Welcome to the SolrCloud example!
Starting up 2 Solr nodes for your example SolrCloud cluster.
Creating Solr home directory d:\solr-5.5.0\example\cloud\node1\solr
Cloning d:\solr-5.5.0\example\cloud\node1 into
   d:\solr-5.5.0\example\cloud\node2
Starting up Solr on port 8983 using command:
d:\solr-5.5.0\bin\solr.cmd start -cloud -p 8983 -s "d:\solr-5.5.0\example\cloud\
node1\solr"
Waiting up to 30 to see Solr running on port 8983
Starting up Solr on port 7574 using command:
d:\solr-5.5.0\bin\solr.cmd start -cloud -p 7574 -s "d:\solr-5.5.0\example\cloud\
node2\solr" -z localhost:9983
Started Solr server on port 8983. Happy searching!
Waiting up to 30 to see Solr running on port 7574
Started Solr server on port 7574. Happy searching!
Connecting to ZooKeeper at localhost:9983 ...
Uploading d:\solr-5.5.0\server\solr\configsets\data_driven_schema_configs\conf f
or config gettingstarted to ZooKeeper at localhost:9983
Creating new collection 'gettingstarted' using command:
http://localhost:8983/solr/admin/collections?action=CREATE&name=gettingstarted&n
umShards=2&replicationFactor=2&maxShardsPerNode=2&collection.configName=gettings
tarted
{
  "responseHeader":{
    "status":0,
    "QTime":6080},
  "success":{"":{
      "responseHeader":{
        "status":0,
        "QTime":5576},
      "core":"gettingstarted_shard2_replica2"}}}
Enabling auto soft-commits with maxTime 3 secs using the Config API
POSTing request to Config API: http://localhost:8983/solr/gettingstarted/config
{"set-property":{"updateHandler.autoSoftCommit.maxTime":"3000"}}
Successfully set-property updateHandler.autoSoftCommit.maxTime to 3000
SolrCloud example running, please visit: http://localhost:8983/solr

成功启动之后，打开浏览器，访问http://localhost:8983/solr 即可看到Solr的控制台界面。

如果不想使用默认的端口**8983**，那么使用如下命令通过`-p`参数指定端口号同样可以启动**Solr Server**，访问http://localhost:8080/solr/index.html#/ 就可以看到主控界面了
```shell
D:\solr-5.5.0\bin>solr.cmd start -cloud -p 8080 -s "D:\solr-5.5.0\example\cloud\
node1\solr"
Backing up D:\solr-5.5.0\example\cloud\node1\solr\..\logs\solr.log
移动了         1 个文件。
Backing up D:\solr-5.5.0\example\cloud\node1\solr\..\logs\solr_gc.log
移动了         1 个文件。
Waiting up to 30 to see Solr running on port 8080
Started Solr server on port 8080. Happy searching!
```
或者直接修改**jetty**的配置文件，修改`D:/solr-5.5.0/server/etc`目录下`jetty-http.xml`中的**8983**改为自己希望的端口号即可。


参考文献
【1】http://blog.csdn.net/luenxin/article/details/50838895
