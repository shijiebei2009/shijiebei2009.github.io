title: Gradle使用扩展属性管理依赖版本号
date: 2017-05-09 21:15:19
tags: [Gradle]
categories: Programming Notes

---

#### Maven预设变量
使用过`Maven`的人应该都知道，我们在`Maven`项目中添加依赖的一般性做法。就是打开`pom.xml`文件，在`<dependencies>`节点下添加
```xml
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-core</artifactId>
    <version>5.5.0</version>
</dependency>
```
包含坐标和版本号的内容，那么在`Java`类文件中，就可以引用`Lucene`包中的各种类了。但是要注意一点，这里面的版本号是以硬编码的形式存在，作为一个合格的软件开发者，要尽量在你的代码中避免硬编码的情况。为什么呢？比如我需要依赖其它的`Lucene`模块，那么`pom.xml`中添加内容如下：
```xml
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-analyzers-common</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-queryparser</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-highlighter</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-core</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-queries</artifactId>
    <version>5.5.0</version>
</dependency>
```
假设经年累月，项目需要升级，`Lucene`的新版本也已发布，那么是不是需要手动修改每一行`<version>5.5.0</version>`呢？这还只是依赖几个模块的问题，假设你依赖成百上千个模块，其版本号都需要升级，是不是觉得想抽当初的自己呢？其实在`Maven`中这种情况很好解决、就是利用预设变量。

```xml
<properties>
    <version.lucene>6.0.0</version.lucene>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-analyzers-common</artifactId>
        <version>${version.lucene}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-queryparser</artifactId>
        <version>${version.lucene}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-highlighter</artifactId>
        <version>${version.lucene}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-core</artifactId>
        <version>${version.lucene}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-queries</artifactId>
        <version>${version.lucene}</version>
    </dependency>
</dependencies>
```
那么以后再遇到项目升级的情况，只需要手动修改`<version.lucene>6.0.0</version.lucene>`一行代码即可搞定，所有引用到该版本变量的依赖都自动升级，这样来管理依赖，是不是很哈皮呢？

#### Gradle单模块
同样作为后起之秀的`Gradle`如何优雅地解决类似问题呢？硬编码的写法如下
```xml
dependencies {
    compile "org.apache.lucene:lucene-core:5.5.0"
}
```
优雅的写法①如下，打开`build.gradle`文件
```xml
ext {
    luceneVersion = '6.5.0'
}
dependencies {
    compile "org.apache.lucene:lucene-core:$luceneVersion"
}
```
一定要注意包含`$`符号时，要用双引号，我就因用单引号在这上吃过亏。在`Gradle`中单引号和双引号都是合法的，但是略有不同。单引号中的内容严格对应`Java`中的`String`，不对`$`符号进行转义。双引号的内容则和脚本语言处理有点像，如果字符中有`$`号的话，则它会先对`$`表达式求值。在`Gradle`中，其实还有三引号的情形，这代表什么呢？三引号中的字符串支持任意换行，比如
```gradle
   def multieLines = ''' begin
     line  1
     line  2
     line  3
     end '''
```

除了①，还有优雅的写法②，使用字典类型，修改`build.gradle`文件
```xml
ext {
    javaSourceCompatibility = '1.8'
    libVersions = [
            junit : '4.12',
            lucene: '6.5.0',
            guava : '20.0'
    ]
}
dependencies {
    compile "junit:junit:$libVersions.junit"
    compile "com.google.guava:guava:$libVersions.guava"
}
```

#### Gradle多模块
当在一个根项目下有多个子模块，那么一种简单的做法是在每个子模块中都定义`ext`代码块，声明需要使用到的版本号变量，这样做当然可以。但是当遇到需要升级版本号的情况时，需要手动修改所有的子模块，其实还有更优雅的解决方案。就是在根项目中定义`ext`代码块，打开根项目的`build.gradle`定义
```xml
ext {
    luceneVersion = '6.5.0'
}
```
然后在每个子模块的`build.gradle`中都可以直接引用之，方式如下
```xml
dependencies {
    compile "junit:junit:$rootProject.libVersions.junit"
}
```
这样，当需要升级版本号的时候，只需要升级根项目中的变量即可，所用子模块的版本号会自动升级。

#### 消除Gradle编译警告
通过Gradle编译项目过程中，有时会报如下警告信息

>注: xxx.java使用或覆盖了已过时的 API。
注: 有关详细信息, 请使用 -Xlint:deprecation 重新编译。
注: xxx.java使用了未经检查或不安全的操作。
注: 有关详细信息, 请使用 -Xlint:unchecked 重新编译。

警告不是Error，虽然不影响编译，但是看着总是不舒服，所以想办法消除警告信息。根据StackOverflow上的[问答](http://stackoverflow.com/questions/18689365/how-to-add-xlintunchecked-to-my-android-gradle-based-project)，在`build.gradle`中添加如下配置即可消除警告。
```xml
allprojects {
    gradle.projectsEvaluated {
        tasks.withType(JavaCompile) {
            options.compilerArgs << "-Xlint:unchecked" << "-Xlint:deprecation"
        }
    }
}
```