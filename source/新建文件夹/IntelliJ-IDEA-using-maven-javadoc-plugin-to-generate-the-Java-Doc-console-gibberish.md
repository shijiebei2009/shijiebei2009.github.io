title: 'IntelliJ IDEA使用maven-javadoc-plugin生成Java Doc控制台乱码'
date: 2016-05-07 08:19:40
tags: [IntelliJ]
categories: Programming Notes

---
####问题重现
在使用**IDEA**生成**Java Doc**的过程中，发现**IDEA**控制台乱码，作为有轻微代码强迫症的我来说，这是不可忍受的，需要鼓捣一番。先上**pom.xml**中的**javadoc**插件配置

```xml
<!--配置生成Javadoc包-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>2.10.3</version>
    <configuration>
        <encoding>UTF-8</encoding>
        <aggregate>true</aggregate>
        <charset>UTF-8</charset>
        <docencoding>UTF-8</docencoding>
    </configuration>
    <executions>
        <execution>
            <id>attach-javadocs</id>
            <phase>package</phase>
            <goals>
                <goal>jar</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
在运行**mvn clean package**命令进行打包之后，控制台会打印出如下信息，可以看到在使用**javadoc**插件的过程中，控制台输出乱码
>[INFO] --- maven-javadoc-plugin:2.10.3:jar (attach-javadocs) @ lucene ---
[INFO] 
���ڼ���Դ�ļ�D:\Multi-module-project\Lucene\src\main\java\AnalyzerDemo.java...
���ڼ���Դ�ļ�D:\Multi-module-project\Lucene\src\main\java\BaiduAPI.java...
���ڼ���Դ�ļ�D:\Multi-module-project\Lucene\src\main\java\CustomQueryParser.java...
...
...

####解决办法
在**IDEA**中，打开**File | Settings | Build, Execution, Deployment | Build Tools | Maven | Runner**在**VM Options**中添加`-Dfile.encoding=GBK`，切记一定是**GBK**。即使用**UTF-8**的话，依然是乱码，这是因为**Maven**的默认平台编码是**GBK**，如果你在命令行中输入`mvn -version`的话，会得到如下信息，根据**Default locale**可以看出
>Maven home:...
Java version:...
Java home:...
**Default locale: zh_CN, platform encoding: GBK**
...
...

再次运行**mvn clean package**，控制台输出一切正常
>[INFO] --- maven-javadoc-plugin:2.10.3:jar (attach-javadocs) @ lucene ---
[INFO] 
正在加载源文件D:\Multi-module-project\Lucene\src\main\java\AnalyzerDemo.java...
正在加载源文件D:\Multi-module-project\Lucene\src\main\java\BaiduAPI.java...
正在加载源文件D:\Multi-module-project\Lucene\src\main\java\CustomQueryParser.java...
...
...