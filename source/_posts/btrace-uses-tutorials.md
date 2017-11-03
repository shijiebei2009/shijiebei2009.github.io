---
title: BTrace 使用教程
date: 2017-09-22 21:48:40
tags: [BTrace]
categories: Programming Notes

---
### 背景
在日常开发中，有一些常见的环境，比如Dev、UAT、预发、生产等，当然并不是每个公司都是这样。有时候开发环境一切正常，但是到线上的UAT环境或预发等等会出现各种问题，那么你是不是经常需要进行本地修改代码、提交、编译、打包、上传、运行、查看日志等这一系列步骤呢？这种方式不仅低效、繁琐而且容易引入诸多不可控的因素，比如你在任意一个环节出现问题，可能都会影响到程序最终的运行结果。而如果能有一种神器，可以对正在运行的程序，进行动态追踪、错误诊断、性能剖析等，是不是无形中为你延长了生命呢？如果你之前不知道也就罢了，然而如果你看到这里了，却还不学习的话，就是你自己的锅了。

### Java运行时追踪工具
常见的动态追踪工具有[BTrace](https://github.com/btraceio/btrace)、[HouseMD](https://github.com/CSUG/HouseMD)（该项目已经停止开发）、[Greys-Anatomy](https://github.com/oldmanpushcart/greys-anatomy)（国人开发，个人开发者）、[Byteman](http://byteman.jboss.org/)（JBoss出品），注意Java运行时追踪工具并不限于这几种，但是这几个是相对比较常用的，本文主要介绍BTrace。

### BTrace
#### BTrace简介
BTrace是SUN Kenai云计算开发平台下的一个开源项目，旨在为java提供安全可靠的动态跟踪分析工具。先看一下BTrace的官方定义
>BTrace is a safe, dynamic tracing tool for the Java platform. BTrace can be used to dynamically trace a running Java program (similar to DTrace for OpenSolaris applications and OS). BTrace dynamically instruments the classes of the target application to inject tracing code ("bytecode tracing")

简洁明了，大意是一个Java平台的安全的动态追踪工具。可以用来动态地追踪一个运行的Java程序。BTrace动态调整目标应用程序的类以注入跟踪代码（“字节码跟踪”）。

动手之前再了解一下BTrace的主要术语
- **Probe Point**: "location" or "event" at which a set of tracing statements are executed. Probe point is "place" or "event" of interest where we want to execute some tracing statements.（探测点，就是我们想要执行一些追踪语句的地方或事件）
- **Trace Actions or Actions**: Trace statements that are executed whenever a probe "fires".（当探测触发时执行追踪语句）
- **Action Methods**: BTrace trace statements that are executed when a probe fires are defined inside a static method a class. Such methods are called "action" methods.（当在类的静态方法中定义了探测触发时执行的BTrace跟踪语句。这种方法被称为“操作”方法。）

#### 安装BTrace
目前，BTrace已经托管在Github上了，主页在[这里](https://github.com/btraceio/btrace)，下载地址在[这里](https://github.com/btraceio/btrace/releases)，目前最新版本是`V1.3.9`。新建环境变量`BTRACE_HOME`值为`E:/btrace-bin-1.3.9`，然后编辑`Path`变量，在值的末尾追加`;%BTRACE_HOME%/bin`即可，验证是否安装成功，打开cmd，输入btrace，显示如下则证明配置成功
>Usage: btrace <options> <pid> <btrace source or .class file> <btrace arguments>
where possible options include:
  --version             Show the version
  -v                    Run in verbose mode
  -o <file>             The path to store the probe output (will disable showing the output in console)
  -u                    Run in trusted mode
  -d <path>             Dump the instrumented classes to the specified path
  -pd <path>            The search path for the probe XML descriptors
  -classpath <path>     Specify where to find user class files and annotation processors
  -cp <path>            Specify where to find user class files and annotation processors
  -I <path>             Specify where to find include files
  -p <port>             Specify port to which the btrace agent listens for clients
  -statsd <host[:port]> Specify the statsd server, if any

根据上面的提示，btrace使用起来很简单，而且官方提供了一个简易的使用指南，在解压下载的压缩包中`E:/btrace-bin-1.3.9/docs`下有`usersguide.html`，用浏览器打开即可。BTrace支持四种方式的注解，分别是
- **Method Annotations**
 - @com.sun.btrace.annotations.OnMethod
 - @com.sun.btrace.annotations.OnTimer
 - @com.sun.btrace.annotations.OnError
 - @com.sun.btrace.annotations.OnExit
 - @com.sun.btrace.annotations.OnEvent
 - @com.sun.btrace.annotations.OnLowMemory
 - @com.sun.btrace.annotations.OnProbe
- **Argument Annotations**
 - @com.sun.btrace.annotations.Self
 - @com.sun.btrace.annotations.Return
 - @com.sun.btrace.annotations.CalledInstance
 - @com.sun.btrace.annotations.CalledMethod
- **Field Annotations**
 - @com.sun.btrace.annotations.Export
 - @com.sun.btrace.annotations.Property
 - @com.sun.btrace.annotations.TLS
- **Class Annotations**
 - @com.sun.btrace.annotations.DTrace
 - @com.sun.btrace.annotations.DTraceRef
 - @com.sun.btrace.annotations.BTrace

关于这些注解的具体解释可以去翻看docs目录下的用户指南，好了，废话不多说，下面简单操练起来。

#### 使用示例
如果是在Maven项目中开发，那么首先需要引入BTrace的Jar包，由于Maven的中央仓库中只有1.x版本的BTrace，并没有高版本的，所以一般的做法是自己编译BTrace源码，将高版本的Jar发布到私服（Nexus）中，为简单起见，此处通过Maven指定依赖本地Jar即可，修改pom.xml
```xml
<dependency>
    <groupId>com.sun.tools.btrace</groupId>
    <artifactId>btrace-agent</artifactId>
    <version>1.3.9</version>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.9/build/btrace-agent.jar</systemPath>
</dependency>
<dependency>
    <groupId>com.sun.tools.btrace</groupId>
    <artifactId>btrace-boot</artifactId>
    <version>1.3.9</version>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.9/build/btrace-boot.jar</systemPath>
</dependency>
<dependency>
    <groupId>com.sun.tools.btrace</groupId>
    <artifactId>btrace-client</artifactId>
    <version>1.3.9</version>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.9/build/btrace-client.jar</systemPath>
</dependency>
```

首先编写想要追踪的示例，如下
```java
import java.util.concurrent.TimeUnit;

public class BTraceOnMethodDemo {
    public static void main(String[] args) {
        try {
            TimeUnit.SECONDS.sleep(15);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("start main method...");
        new Thread(() -> {
            while (true) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```

再编写追踪代码示例，如下
```java
import com.sun.btrace.annotations.BTrace;
import com.sun.btrace.annotations.OnMethod;

import static com.sun.btrace.BTraceUtils.println;

@BTrace
public class Tracer {
    @OnMethod(clazz = "java.lang.Thread", method = "start")
    public static void onThreadStart() {
        println("tracing method start");
    }
}
```

注意点
- Tracer类中的println方法并不是JDK中的方法，而是BTraceUtils中的静态方法
- 在追踪代码，原则上只能使用BTraceUtils中的静态方法，如果想要使用JDK中的方法，那么在命令行中需要使用`-cp`指定依赖的Jar
- OnMethod这个注解只会在方法启动的时候被触发，如果该方法已经启动，再运行追踪代码是无法触发的，所以在示例中，先休息了15s钟

运行BTraceOnMethodDemo，打开cmd，到Tracer类所在的目录下，运行
>D:/Demo/src/main/java>jps
39808 Launcher
15480 RemoteMavenServer
40280 Jps
41880 AppMain
4764
D:/Demo/src/main/java>btrace -v 41880 Tracer.java
DEBUG: assuming default port 2020
DEBUG: assuming default classpath '.'
DEBUG: compiling Tracer.java
DEBUG: compiled Tracer.java
DEBUG: attaching to 41880
DEBUG: checking port availability: 2020
DEBUG: attached to 41880
DEBUG: loading E:\btrace-bin-1.3.9\build\btrace-agent.jar
DEBUG: agent args: port=2020,statsd=,debug=true,bootClassPath=.,systemClassPath=
C:\Program Files\Java\jdk1.8.0_45\jre/../lib/tools.jar,probeDescPath=.
DEBUG: loaded E:\btrace-bin-1.3.9\build\btrace-agent.jar
DEBUG: registering shutdown hook
DEBUG: registering signal handler for SIGINT
DEBUG: submitting the BTrace program
DEBUG: opening socket to 2020
DEBUG: setting up client settings
DEBUG: sending instrument command
DEBUG: entering into command loop
DEBUG: received com.sun.btrace.comm.OkayCommand@396f6598
DEBUG: received com.sun.btrace.comm.RetransformationStartNotification@394e1a0f
DEBUG: received com.sun.btrace.comm.OkayCommand@27a5f880
DEBUG: received com.sun.btrace.comm.MessageCommand@53f65459
DEBUG: received com.sun.btrace.comm.MessageCommand@3b088d51
tracing method start

注意要带上`-v`参数，否则控制台看不到任何输出，另外还可以利用`-o`参数将信息输出到指定的文件，运行BTraceOnMethodDemo，打开cmd，到Tracer类的目录下，运行
>D:/Demo/src/main/java>jps
12064 Jps
24560 AppMain
41764 Launcher
15480 RemoteMavenServer
4764
D:/Demo/src/main/java>btrace -o out.csv 24560 Tracer.java

注意这时候out.csv文件时在Tracer.java所在目录的根目录下，也就是在`D:/Demo`下，在当前目录下是找不到的，是不是很变态。找到out.csv打开看看，就是追踪代码的输出内容
>BTrace Log: 17-9-20 下午3:16
tracing method start

好了，整个流程就打通了，剩下的就是自己动手实战吧。此处仅给出一个简单示例，详情可以参看BTrace的用户指南，里面给出了更多详细的示例，只要打开动手一一实战即可。

BTrace虽然功能强大，但是并不完美，这是因为它有着诸多的限制，例如
*   can **not** create new objects.
*   can **not** create new arrays.
*   can **not** throw exceptions.
*   can **not** catch exceptions.
*   can **not** make arbitrary instance or static method calls - only the **public static** methods of **[com.sun.btrace.BTraceUtils](javadoc/com/sun/btrace/BTraceUtils.html)** class may be called from a BTrace program.
*   can **not** assign to static or instance fields of target program's classes and objects. But, BTrace class can assign to it's own static fields ("trace state" can be mutated).
*   can **not** have instance fields and methods. Only **static public void** returning methods are allowed for a BTrace class. And all fields have to be static.
*   can **not** have outer, inner, nested or local classes.
*   can **not** have synchronized blocks or synchronized methods.
*   can **not** have loops (**for, while, do..while**)
*   can **not** extend arbitrary class (super class has to be java.lang.Object)
*   can **not** implement interfaces.
*   can **not** contains assert statements.
*   can **not** use class literals.

#### BTrace命令详解
- **btrace**
功能：用于运行BTrace跟踪程序。
命令格式：
`btrace [-I <include-path>] [-p <port>] [-cp <classpath>] <pid> <btrace-script> [<args>]`
示例：
`btrace -cp build/  1200 AllCalls1.java`
参数含义：
include-path指定头文件的路径，用于脚本预处理功能，可选；
port指定BTrace agent的服务端监听端口号，用来监听clients，默认为2020，可选；
classpath用来指定类加载路径，默认为当前路径，可选；
pid表示进程号，可通过jps命令获取；
btrace-script即为BTrace脚本；btrace脚本如果以.java结尾，会先编译再提交执行。可使用btracec命令对脚本进行预编译。
args是BTrace脚本可选参数，在脚本中可通过`$`和`$length`获取参数信息。

- **btracec**
功能：用于预编译BTrace脚本，用于在编译时期验证脚本正确性。
`btracec [-I <include-path>] [-cp <classpath>] [-d <directory>] <one-or-more-BTrace-.java-files>`
参数意义同btrace命令一致，directory表示编译结果输出目录。

- **btracer**
功能：btracer命令同时启动应用程序和BTrace脚本，即在应用程序启动过程中使用BTrace脚本。而btrace命令针对已运行程序执行BTrace脚本。
命令格式：
`btracer <pre-compiled-btrace.class> <application-main-class> <application-args>`
参数说明：
pre-compiled-btrace.class表示经过btracec编译后的BTrace脚本。
application-main-class表示应用程序代码；
application-args表示应用程序参数。
该命令的等价写法为：
`java -javaagent:btrace-agent.jar=script=<pre-compiled-btrace-script1>[,<pre-compiled-btrace-script1>]* <MainClass> <AppArguments>`

BTrace基本就介绍完了，但是BTrace并不是完美的，比如当你想要追踪一个局部变量的，查看具体值的时候，却无能为力，不仅扼腕叹息，真是天妒英才啊，这么小的一个需求都无法cover？不用着急，后面就介绍一个更加强大的工具，Byteman。