title: Linux下使用java -jar运行可执行jar包的正确方式
date: 2016-05-11 20:38:26
tags: [Java]
categories: Programming Notes

---
####问题来源
一般来说，一个稍微大些的项目都会有一些依赖的**Jar**包，而在将项目部署到服务器的过程中，如果没有持续集成环境的话，也就是说服务器不支持在线编译及打包，那么需要自己上传依赖的**Jar**包，然而可能服务器上已经存在了该项目所依赖的**Jar**包（比如项目修复BUG，重新打包上传，而依赖不变），无需再次上传，此时只需将该项目单独打包，在运行的时候指定**CLASSPATH**即可。

在将**Jar**包部署到服务器上之后，设置**CLASSPATH**环境变量，运行`java -jar ...`命令出现**ClassNotFoundException**异常。之后又试用了诸多其它参数设置CLASSPATH，例如下面几个命令，同样都是报找不到类异常。
```
set CLASSPATH = classpath1;classpath2...
java -classpath ".;D:\mylib\*" -jar jar包 #Windows设置
java -classpath ".:/data/home/mylib/*" -jar jar包 #Linux设置
java -cp ...
java -cp /lib/*
```
关于在**CLASSPATH**参数中使用通配符需要注意
正确方式（冒号代表是Linux，Windows使用分号）
```
java -classpath "lib/*:." my.package.Program
```
不正确方式
```
java -classpath "lib/a*.jar:." my.package.Program
java -classpath "lib/a*:."     my.package.Program
java -classpath "lib/*.jar:."  my.package.Program
java -classpath  lib/*:.       my.package.Program
```

####解决办法
首先你需要知道**Jar**包分为可执行**Jar**和非可执行**Jar**，一个可执行的**Jar**文件是一个自包含的**Java**应用程序，它存储在特别配置的**JAR**文件中，可以由**JVM**直接执行它而无需事先提取文件或者设置类路径。要运行存储在非可执行的**JAR**中的应用程序，必须将它加入到您的类路径中，并用名字调用应用程序的主类。但是使用可执行的**JAR**文件，我们可以不用提取它或者知道主要入口点就可以运行一个应用程序。可执行**JAR**有助于方便发布和执行**Java**应用程序。

对于可执行**Jar**，在运行**java -jar**选项的时候，那么环境变量**CLASSPATH**和在命令行中指定的所有类路径都将被**JVM**忽略，也就是说，对于一个可执行**Jar**，使用**java -classpath**或者**java -cp**或者**set classpath=lib/commons-io-2.4.jar**等等命令指定**CLASSPATH**都是无效的。

对于一个可执行的**JAR**必须通过**MANIFEST.MF**文件的头引用它所需要的所有其他从属**JAR**，引用方式如下
```xml
Class-Path: lib/commons-io-2.4.jar lib/commons-lang3-3.4.jar
```
如果有多个**Jar**包那么相互之间使用空格分隔。**MANIFEST**文件的一般格式如下
```xml
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: wangxu
X-Compile-Target-JDK: 1.7
X-Compile-Source-JDK: 1.7
Created-By: Apache Maven 3.3.3
Build-Jdk: 1.8.0_45
Main-Class: com.yuewen.statistics.report.service.Main
Class-Path: lib/commons-io-2.4.jar lib/commons-lang3-3.4.jar lib/guava-18.0.jar lib/junit-4.10.jar lib/log4j-api-2.0.jar lib/log4j-core-2.0.jar lib/lombok-1.16.4.jar lib/lucene-analyzers-common-5.5.0.jar lib/lucene-analyzers-smartcn-5.5.0.jar lib/lucene-core-5.5.0.jar lib/lucene-grouping-5.5.0.jar lib/lucene-queries-5.5.0.jar lib/lucene-queryparser-5.5.0.jar lib/mysql-connector-java-5.1.38-bin.jar

```
其中**Manifest-Version**表示版本号，一般由**IDE**工具自动生成，在编写**MANIFEST**文件的过程中，有如下注意点
- Main-Class是jar文件的主类，程序的入口
- Class-Path指定需要的jar，多个jar必须要在一行上，多个jar之间以空格隔开，如果引用的jar在当前目录的子目录下，windows下使用\来分割，linux下用/分割
- 文件的冒号后面必须要空一个空格，否则会出错
- 文件的最后一行必须是一个回车换行符，否则也会出错

####多条java jar命令的执行顺序问题
通常情况下，我们会在服务器上配置**shell**脚本去定时调用自己的**Jar**包，但是当**shell**脚本中存在多条`java -jar`命令时，其执行情况是怎么样的呢？是同时并行执行，还是挨个按顺序执行呢？经过测试得出，多条`java -jar`命令是按顺序执行的，并且只有在第一条`java -jar`命令执行完毕后，才会执行下一条`java -jar`命令，依次类推。

参考资料
【1】http://stackoverflow.com/questions/219585/setting-multiple-jars-in-java-classpath/219801#219801
【2】https://www.ibm.com/developerworks/cn/java/j-jar/