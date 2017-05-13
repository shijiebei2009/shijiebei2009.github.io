title: "IntelliJ IDEA中Maven插件无法更新索引之解决办法"
date: 2015-12-09 22:21:33
tags: [IntelliJ]
categories: Programming Notes

---

####Maven的仓库、索引
**中央仓库**：目前来说，[http://repo1.maven.org/maven2/](http://repo1.maven.org/maven2/) 是真正的`Maven`中央仓库的地址，该地址内置在`Maven`的源码中，其它地址包括著名的[ibiblio.org](http://mirrors.ibiblio.org/pub/mirrors/maven2/)，都是镜像。

**索引**：中央仓库带有索引文件以方便用户对其进行搜索，完整的索引文件至2015年12月8日大小约为1.11G，索引每周更新一次。

**本地仓库**：是建立在本地机器上的`Maven`仓库，本地仓库是中央仓库（或者说远程仓库）的一个缓冲和子集，当你构建`Maven`项目的时候，首先会从本地仓库查找资源，如果没有，那么`Maven`会从远程仓库下载到你本地仓库。这样在你下次使用的时候就不需要从远程下载了。如果你所需要的`Jar`包版本在本地仓库没有，而且也不存在于远程仓库，`Maven`在构建的时候会报错，这种情况可能发生在有些`Jar`包的新版本没有在`Maven`仓库中及时更新。Maven缺省的本地仓库地址为`${user.home}/.m2/repository`。也就是说，一个用户会对应的拥有一个本地仓库。当然你可以通过修改`${user.home}/.m2/settings.xml`配置这个地址：
```xml
<settings>
  ···
  <localRepository> D:/java/repository</localRepository>
  ...
</settings>
```
**提交内容**：只要你的项目是开源的，而且你能提供完备的`POM`等信息，你就可以提交项目文件至中央仓库，这可以通过`Sonatype`提供的[开源Maven仓库托管服务](https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide)实现。

####IntelliJ IDEA利用索引实现自动补全
众所周知，由于伟大的中国防火墙，所以在使用IDEA下载Maven仓库索引的时候，要么无法访问，要么就是速度极慢，这对开发人员带来了极大的不便，所以一般公司都用Nexus搭建一个公司内部的私服。同时利用私服更有利于对公司内部开发人员依赖的Jar包版本进行控制。

也许你会问，中央仓库带有索引，为什么本地的IDEA也需要下载索引呢？那么直接看下图你就明白了，如果本地没有下载索引的话，在`pom.xml`文件中添加依赖是得不到任何提示的。
![](http://7xig3q.com1.z0.glb.clouddn.com/maven_after_update_maven_index_add_dependence%20.gif)

####IntelliJ IDEA中Maven插件配置
IntelliJ已经内置了对Maven插件的支持，当然你也可以配置自己的Maven，只需要进入`Settings->Maven->Maven home directory|User settings file|Local repository`配置即可。注意如果使用自己配置的Maven，那么一定要勾选`Override`，否则配置不生效。
![](http://7xig3q.com1.z0.glb.clouddn.com/IntelliJ_plugin_maven_config.png)

####IntelliJ14.1更新索引失败原因
在使用14.1.X版本的IntelliJ时，更新Maven索引出现如下错误[Indexed Maven Repositories - type remore - Error - Idea 14.1.5](https://devnet.jetbrains.com/message/5560886;jsessionid=565FE35134A3F90A560B993435EAC7EF#5560886)，根据该链接内所述原因为：这是IntelliJ14.1.X版本中的一个BUG，并且会在下一个发布版本中进行修复，推荐将IntelliJ升级到版本15。

####使用国内Maven仓库的镜像
鉴于伟大的防火墙，所以推荐使用国内的镜像资源作为Maven中央仓库。推荐使用[开源中国Maven库使用帮助](http://maven.oschina.net/help.html)，配置很简单就不详述了，有两种方式，其一打开**settings.xml**文件，加入
```xml
<mirrors>
    <mirror>
        <id>nexus-osc</id>
        <mirrorOf>*</mirrorOf><!--用一个简单的*号会把所有的仓库地址屏蔽掉-->
        <name>Nexus osc</name>
        <url>http://maven.oschina.net/content/groups/public/</url>
    </mirror>
</mirrors>
```
当然还有第二种方式，就是屏蔽指定的中央仓库，并且还可以加入**OSChina**的第三方镜像仓库或者多个仓库，配置如下
```xml
<mirrors>
    <mirror>
        <id>nexus-osc</id>
        <mirrorOf>central</mirrorOf><!--这里指定只屏蔽central仓库-->
        <name>Nexus osc</name>
        <url>http://maven.oschina.net/content/groups/public/</url>
    </mirror>
    <mirror>
        <id>nexus-osc-thirdparty</id>
        <mirrorOf>thirdparty</mirrorOf>
        <name>Nexus osc thirdparty</name>
        <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
    </mirror>
</mirrors>
```
最后，在执行**Maven**命令的时候，**Maven**还需要安装一些插件包，这些插件包的下载地址也让其指向**OSChina**的**Maven**地址。修改如下所示
```xml
<profile>
     <id>jdk-1.8</id>
     <activation>
         <jdk>1.8</jdk><!--指定JDK版本是1.8时自动激活-->
     </activation>
     <repositories>
         <repository>
            <id>nexus</id>
            <name>local private nexus</name>
            <url>http://maven.oschina.net/content/groups/public/</url>
            <releases>
              <enabled>true</enabled>
            </releases>
            <snapshots>
              <enabled>false</enabled>
            </snapshots>
         </repository>
     </repositories>
     <pluginRepositories>
         <pluginRepository>
            <id>nexus</id>
            <name>local private nexus</name>
            <url>http://maven.oschina.net/content/groups/public/</url>
            <releases>
              <enabled>true</enabled>
            </releases>
            <snapshots>
              <enabled>false</enabled>
            </snapshots>
         </pluginRepository>
     </pluginRepositories>
</profile>
```
另外你也可以下载开源中国提供的官方纯净版[settings.xml](http://maven.oschina.net/static/xml/settings.xml)文件。

####下载Maven仓库的索引
在配置完成之后就可以下载仓库索引了，注意这是一个非常耗时的过程，建议利用晚上或者出去午饭时间下载。下载过程及下载完成之后状态如下图所示。本次下载整体耗时在一个小时左右。
![](http://7xig3q.com1.z0.glb.clouddn.com/IntelliJ_maven_download_local_index.png)
另外我在思考既然下载一次这么麻烦，那么下载下来的索引存放在哪里呢？我能否将其拷贝到其他机器重复利用呢？于是经过一番搜索我发现了索引的存放位置，并且将其打包拷贝到其他机器的同样位置，但未做测试，不知能否重复利用，如有网友测试完毕，可以告诉我，感谢之。
![](http://7xig3q.com1.z0.glb.clouddn.com/IntelliJ_maven_index_location_detailed.png)

####利用本地Tomcat作为索引下载服务器
- 首先下载如下两个文件：
http://repo1.maven.org/maven2/.index/nexus-maven-repository-index.properties
http://repo1.maven.org/maven2/.index/nexus-maven-repository-index.gz
- 启动一个Apache Tomcat服务器，在其根目录下建立一个`/maven2/.index`的虚拟目录（注意：如果你使用的是Windows系统，可能无法建立`.index`件夹，必须使用DOS命令：`mkdir .index`），把上述两个文件拷贝至该虚拟目录下
- 编辑`C:/WINDOWS/system32/drivers/etc/hosts`文件，在文件中加入:
`127.0.0.1    repo1.maven.org`
注意：`127.0.0.1`为步骤2的`Apache Tomcat`服务器IP地址。
- 在IDEA的maven插件中更新索引
- 移除步骤3中在hosts文件中添加的内容

**备注**：其实该解决办法的总体思路就是先将索引文件整体下载，然后利用本地的Tomcat作为服务器，再从Tomcat上更新索引。

最后如果你想自己配置一个私服，可以参考[Maven仓库管理之Nexus](http://my.oschina.net/aiguozhe/blog/101537?fromerr=kOXkYkdf)。

####开源中国镜像存在的问题
- 开源中国镜像不是很稳定，有时候很快下载完成有时候一直处于`Resolving dependencies of ... `状态而无法下载
- 在配置了开源中国第三方库镜像之后，发现一个问题，该库内容更新不及时，很多第三方库中的Jar包版本都非常陈旧。
- 开源中国的中央仓库与第三方库中存在很多交叉的情况，也就是说中央仓库包括了第三方库中的内容，而且在下载jar文件的时候，默认就是直接从开源中国的中央仓库镜像下载，而不是开源中国的第三方仓库镜像下载。
- 我给出的建议是，如无必要，移除开源中国的第三方库镜像地址，移除的内容如下
```xml
<mirror>
      <id>nexus-osc-thirdparty</id>
      <mirrorOf>thirdparty</mirrorOf>
      <name>Nexus osc thirdparty</name>
      <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
</mirror>
```
- 针对以上问题，有时候还是需要从国外Maven官方的仓库下载，方法是只需要修改`settings.xml`文件为官方默认版本即可。现将Maven默认`settings.xml`贴出
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>D:/apache-maven-3.3.1/repository</localRepository>
  </pluginGroups>
  <proxies>
  </proxies>
  <servers>
  </servers>
  <mirrors>
  </mirrors>
  <profiles>
  </profiles>
</settings>
```
