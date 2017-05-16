title: 构建系统Gradle备忘录
date: 2016-11-03 20:52:36
tags: [Gradle]
categories: Programming Notes

---

- Gradle是什么？
Gradle是一个集合了Maven和Ant优点的构建工具，据说要取代Maven，不置可否。

- 什么是projects和tasks？
每一个构建都是由一个或多个projects构成的。一个project到底代表什么取决于你想用Gradle做什么。每一个project是由一个或多个tasks构成的，一个task代表一些更加细化的构建。可能是编译一些classes，创建一个JAR，生成javadoc或者生成某个目录的压缩文件。

- 经常用的`gradle -q`，其中`-q`是干什么的？
`-q`代表quiet模式，它不会生成Gradle的日志信息(log messages)，所以用户只能看到tasks的输出，它使得输出更加清晰。

- 如何定义一个task？
在build.gradle中
```groovy
task hello {
    doLast {
        println("hello world!")
    }
}
```
  其中task名称叫hello，hello的action叫doLast。

- 如何快捷的定义一个task？
```groovy
task hello << {
    println("hello world!")
}
```
  其中doLast被替换成了`<<`

- 任务之间如何相互依赖？
```groovy
task first << {
    println("first!")
}
task after(dependsOn: first) << {
    println("after!")
}
```
  **gradle -q after**输出：
```groovy
first!
after!
```

- 动态的创建任务
```groovy
4.times {
    counter ->
        task "task$counter" << {
            println("I'm task number $counter!")
        }
}
```
  这里动态的创建了task0，task1，task2，task3
  **gradle -q task3**输出：`I'm task number 3!`
  如果使用`task0.dependsOn task2, task3`指定任务依赖关系，那么**gradle -q task0**输出：
```groovy
I'm task number 2!
I'm task number 3!
I'm task number 0!
```

- 短标记法
有一个短标记`$`可以访问一个存在的任务，也就是说每个任务都可以作为构建脚本的属性。
```groovy
task first << {
    println("first!")
}
task after << {
    println("$first.name")
}
```
  **gradle -q after**输出：`first`
  其中name是任务的默认属性，代表当前任务的名称。

- 自定义任务属性
```groovy
task first {
    ext.myProperty = "value"
}
task after << {
    println(first.myProperty)
}
```
  **gradle -q after**输出：`value`

- 默认任务
```groovy
defaultTasks 'validate', 'run'
task validate << {
    println("Default Validating")
}
task run << {
    println("Default Running")
}
```
  **gradle -q**输出：
```groovy
Default Validating
Default Running
```

- 列出可以执行的任务
使用`gradle tasks`来列出项目的所有任务

- Java插件下常用任务
使用`apply plugin: 'java'`之后，Java插件会向你的项目里加入许多任务，常用的如下
**build**任务，它会建立你的项目；
**clean**任务，删除build生成的目录和所有生成的文件；
**assemble**任务，编译并打包你的代码，但是并不运行单元测试。其他插件会在这个任务里加入更多的东西。举个例子，如果你使用War插件，这个任务将根据你的项目生成一个 WAR文件；
**check**任务，编译并测试你的代码。其他的插件会加入更多的检查步骤。举个例子，如果你使用checkstyle插件，这个任务将会运行Checkstyle来检查你的代码。

- 查看哪些属性是可用的
使用`gradle properties`命令来列出项目的所有属性。

- 发布Jar文件
```groovy
uploadArchives {
    repositories {
        flatDir {
            dirs 'repos'
        }
    }
}
```
  运行`gradle uploadArchives`命令来发布Jar文件。

- 多项目构建
目录结构为
```groovy
multiproject/
  api/
  services/webservice/
  services/shared/
  shared/
```
  则在项目根目录下的settings.gradle中添加`include "shared", "api", "services:webservice", "services:shared"`

- 多项目构建-项目之间的依赖
假设api项目的构建需要用到shared项目的Jar文件，那么在api/build.gradle中加入
```groovy
dependencies {
    compile project(':shared')
}
```

- 依赖配置
**compile**：用来编译项目源代码的依赖。
**runtime**：在运行时被生成的类使用的依赖。默认的，也包含了编译时的依赖。
**testCompile**：编译测试代码的依赖。默认的，包含生成的类运行所需的依赖和编译源代码的依赖。
**testRuntime**：运行测试所需要的依赖。默认的，包含上面三个依赖。

- 构建一个WAR文件
在build.gradle中加入`apply plugin: 'war'`。这个插件也会在你的项目里加入Java插件，运行gradle build将会编译，测试和创建项目的WAR文件。

- 运行Web应用
在build.gradle中加入`apply plugin: 'jetty'`。由于Jetty plugin继承自War plugin。使用gradle jettyRun命令将会把你的工程启动部署到jetty容器中。调用gradle jettyRunWar命令会打包并启动部署到jetty容器中。

- 多任务调用
你可以以列表的形式在命令行中一次调用多个任务，它们所依赖的任务也会被调用。这些任务只会被调用一次。
```groovy
task last << {
    println("LAST")
}
task first(dependsOn: last) << {
    println("FIRST")
}
```
  **gradle -q first last first**和**gradle -q first**命令的输出结果是一样的：
```groovy
LAST
FIRST
```

- 排除任务
你可以用命令行选项`-x`来排除某些任务，例如`gradle dist -x test`命令会排除test任务。即使test任务是dist任务的依赖，test也会被排除，同时test任务的依赖任务也都会被排除，而被test和其他任务同时依赖的任务则不会被排除。

- 失败后继续执行构建
默认情况下，只要有任务调用失败，Gradle就会中断执行。这可能会使调用过程更快，但那些后面隐藏的错误就没有办法发现了。所以你可以使用`--continue`选项在一次调用中尽可能多的发现所有问题。采用`--continue`选项，Gralde会调用每一个任务以及它们依赖的任务。而不是一旦出现错误就会中断执行，所有错误信息都会在最后被列出来。但是一旦某个任务执行失败，那么所有依赖于该任务的子任务都不会被调用。

- 选择执行构建
调用gradle命令时，默认情况下总是会构建当前目录下的文件，可以使用`-b`参数选择其他目录的构建文件，并且当你使用此参数时`settings.gradle`将不会生效。`-b`参数用以指定脚本具体所在位置，格式为`dirpwd/build.gradle`，`-p`参数用以指定脚本目录即可。

- 项目列表
执行`gradle projects`命令会为你列出子项目名称列表。

- 任务列表
执行`gradle tasks`命令会列出项目中所有任务。当然你也可以用`--all`参数来收集更多任务信息。这会列出项目中所有任务以及任务之间的依赖关系。

- 获取任务具体信息
执行`gradle help --task someTask`可以显示指定任务的详细信息，或者多项目构建中相同任务名称的所有任务的信息。

- 获取依赖列表
执行`gradle dependencies`命令会列出项目的依赖列表，所有依赖会根据任务区分，以树型结构展示出来。还可以通过`--configuration`参数来查看指定构建任务的依赖情况。例如`gradle -q api:dependencies --configuration testCompile`。

- 查看特定依赖
执行`gradle dependencyInsight`命令可以查看指定的依赖。例如`gradle -q webapp:dependencyInsight --dependency groovy --configuration compile`。

- 获取项目属性列表
执行`gradle properties`可以获取项目所有属性列表。

- 构建日志
`--profile`参数可以收集一些构建期间的信息并保存到`build/reports/profile`目录下。并且会以构建时间命名这些文件。

- Gradle图形界面
为了辅助传统的命令行交互，Gradle还提供了一个图形界面。我们可以使用Gradle命令中`--gui`选项来启动它。注意：这个命令执行后会使得命令行一直处于封锁状态，直到我们关闭图形界面。如果是在`*nix`系统下，则可以通过加上“&”让它在后台执行：`gradle --gui&`，此命令对Windows系统无效。

- 标准项目属性
 Project对象提供了一些常用的标准属性如下

>|Name   | Type  | Default Value|
 | ------------ | ------------ | ------------ |
 |project   | Project  |Project 实例对象|
 |name   |String   |项目目录的名称|
 |path|String|项目的绝对路径|
 |description|String|项目描述|
 |projectDir|File|包含构建脚本的目录|
 |build|File|projectDir/build|
 |group|Object|未具体说明|
 |version|Object|未具体说明|
 |ant|AntBuilder|Ant实例对象|

- 声明变量
在Gradle构建脚本中有两种类型的变量可以声明：局部变量(local)和扩展属性(extra)。局部变量使用关键字def来声明，其只在声明它的地方可见，扩展属性使用ext声明，例如`ext.purpose = null`，另外，使用ext扩展块可以以此添加多个属性。
```groovy
ext {
    springVersion = "5.0.0"
    emailNotification = "build@master.org"
}
```

- Java插件-依赖配置

>|名称   |扩展   |被使用时运行的任务|含义|
 | ------------ | ------------ | ------------ | ------------ |
 |compile   |-    |compileJava |编译时的依赖|
 |runtime    |compile  |- |运行时的依赖|
 |testCompile|compile  | compileTestJava| 编译测试所需的额外依赖|
 |testRuntime|runtime  | test |仅供运行测试的额外依赖|
 |archives   | -  | uploadArchives| 项目产生的信息单元（如:jar包）|
 |default    |runtime |-| 使用其他项目的默认依赖项，包括该项目产生的信息单元以及依赖|

- gradle利用maven的本地仓库
有的人是从maven转向gradle，那么这时maven已经在本地仓库中缓存了大量的jar文件，如何让gradle利用maven的本地仓库呢？首先在`build.gradle`配置
```groovy
repositories {
    mavenLocal()
}
```
  这时，gradle会查找maven的本地仓库地址，查找顺序如下，可以参考[在线文档](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.RepositoryHandler.html)

  ```groovy
  1. The value of system property 'maven.repo.local' if set;
  2. The value of element <localRepository> of ~/.m2/settings.xml if this file exists and element is set;
  3. The value of element <localRepository> of $M2_HOME/conf/settings.xml (where $M2_HOME is the value of the environment variable with that name) if this file exists and element is set;
  4. The path ~/.m2/repository.
  ```

  例如我的maven配置文件位置在`D:/apache-maven-3.3.3/conf/settings.xml`，并且指定了`<localRepository>D:/apache-maven-3.3.3/repo</localRepository>`那么gradle就会找到该位置

- IDEA中如何设置gradle的本地仓库地址
在IDEA中打开Settings || Build, Execution, Deployment || Build Tools || Gradle，修改Service directory path，该值为Gradle的默认缓存目录中，比如填`D:/apache-maven-3.3.3/repo`，这样gradle就会将jar下载到该路径下，例如我的IDEA中的gradle下载的jar地址为：`D:/apache-maven-3.3.3/repo/caches/modules-2/files-2.1`，这样就可以将maven的本地仓库和gradle的本地仓库合二为一了。顺便在Gradle VM options中填`-Dfile.encoding=UTF-8`可以解决一些中文乱码问题。

 前提是IDEA中的maven已经配置好了，同样在Settings || Build, Execution, Deployment || Build Tools || Maven中配置
 ```groovy
 Maven home directory: D:/apache-maven-3.3.3
 User settings file: D:/apache-maven-3.3.3/conf/settings.xml
 Local repository: D:/apache-maven-3.3.3/repo
 ```
