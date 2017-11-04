---
title: Maven引入本地依赖Jar到可执行Jar包中
date: 2017-06-13 22:17:24
tags: [Maven]
categories: Programming Notes

---

在Maven中，默认地，是不会将依赖的Jar包打入可执行Jar包的，如果需要将依赖打入可执行Jar包，需要在`pom`中添加`maven-assembly-plugin`插件，这个很容易实现，但是在正规开发中不推荐这样使用，为什么？因为稍微大型一些的项目都至少有几十个依赖项，而每次打包都将这些Jar包打入可执行Jar，使得最后生成的可执行Jar体积非常大。标准的做法是，将所有的依赖Jar包都打入lib目录中，而在可执行Jar的`MANIFEST.MF`中指定lib路径即可。这也很容易实现，并不是本文的重点，本文的重点是如何将不在Maven中央仓库中的Jar包，或者说依赖本地的Jar包打入可执行Jar，并更新`MANIFEST.MF`文件。

例如在我的Maven项目中，需要依赖本地Jar，首先将依赖的Jar复制到`src/main/resources/lib`目录下，引用如下
```xml
<dependency>
    <groupId>com.yuewen</groupId>
    <artifactId>lucene</artifactId>
    <version>1.0.0-SNAPSHORT</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/lib/lucene-1.0.0-SNAPSHORT.jar</systemPath>
</dependency>
```

这里的scope只能是system范围，systemPath属性指定Jar包的路径。

下一步将所有依赖的Jar包打入lib目录中，方式如下
```xml
<!--将依赖的资源全部打入lib目录-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <configuration>
        <outputDirectory>${project.build.directory}/lib</outputDirectory>
        <excludeTransitive>false</excludeTransitive>
        <stripVersion>false</stripVersion>
    </configuration>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
                <excludeTransitive>false</excludeTransitive>
                <stripVersion>false</stripVersion>
            </configuration>
        </execution>
    </executions>
</plugin>
```

至此，你在Maven项目中，依赖的所有Jar都被打入到`target/lib`目录下了，剩下的关键一步就是如何添加`MANIFEST.MF`文件了。在pom中添加如下插件

```xml
<!--打包插件，在Jar包中添加Class-Path和Main-Class-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <archive>
            <!--使用自己的Manifest文件，运行正常-->
            <!--<manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>-->
            <!--使用插件添加的Manifest文件，运行正常，一定要注意Manifest中jar包名称和lib文件夹下jar包名称版本号后缀等一定要一致，否则找不到依赖jar，此处有坑-->
            <manifest>
                <addClasspath>true</addClasspath>
                <!--指定依赖资源路径前缀-->
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>cn.codepub.maven.test.Main</mainClass>
            </manifest>
            <!--可以把依赖本地系统的Jar包加入Manifest文件中-->
            <manifestEntries>
                <Class-Path>lib/lucene-1.0.0-SNAPSHORT.jar</Class-Path>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

运行`mvn clean package`执行打包，完成之后，将包含依赖资源的lib目录和可执行Jar放在同一级目录下即可，这样在运行`java -jar xxx.jar`的时候，可执行Jar包可以准确地找到依赖Jar包。并且以这种方式打出来的可执行Jar体积非常小，一般都是几百KB而已。完整的`MANIFEST.MF`文件如下所示

```xml
Manifest-Version: 1.0
Built-By: wangxu
Class-Path: lib/lombok-1.16.12.jar lib/guava-20.0.jar lib/log4j-api-2.
 7.jar lib/log4j-core-2.7.jar lib/lucene-1.0.0-SNAPSHORT.jar
Created-By: Apache Maven 3.3.3
Build-Jdk: 1.8.0_45
Main-Class: cn.codepub.maven.test.Main
```