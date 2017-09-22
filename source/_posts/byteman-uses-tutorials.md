---
title: Byteman 使用教程
date: 2017-09-22 21:50:38
tags: [Byteman]
categories: Programming Notes

---

### Byteman简介
Byteman由JBoss出品，JBoss大家应该都熟悉，顶顶大名的应用服务器JBoss也出自其手。Byteman的代码插入能力相比BTrace而言更强，似乎可以在代码中任意的位置插入我们的跟踪代码（当然，你可能需要对Java代码生成、字节码技术有一定的了解），以及访问当前方法中变量的能力（包括方法参数、局部变量、甚至于调用其它函数的参数值、返回值等），而BTrace在这方面的能力要弱很多。

### 安装Byteman
首先去[官网](http://byteman.jboss.org/downloads.html)下载最新的压缩包，解压，配置环境变量，开始操练，老熟悉了。新建`BYTEMAN_HOME`值是`E:\byteman-3.0.10`，编辑`Path`环境变量，在末尾添加`;%BYTEMAN_HOME%/bin`，打开cmd，输入`bminstall`验证一下
>usage: bminstall [-p port] [-h host] [-b] [-s] [-m] [-Dname[=value]]* pid
  pid is the process id of the target JVM
  -h host selects the host name or address the agent listener binds to
  -p port selects the port the agent listener binds to
  -b adds the byteman jar to the bootstrap classpath
  -s sets an access-all-areas security policy for the Byteman agent code
  -m activates the byteman JBoss modules plugin
  -Dname=value can be used to set system properties whose name starts with "org.jboss.byteman."
  expects to find a byteman agent jar and byteman JBoss modules plugin jar (if -m is indicated) in BYTEMAN_HOME

### 使用示例（读取局部变量）
为什么选择这个示例呢？因为BTrace无法追踪到局部变量的值，那么为了显示Byteman的强大，必须让它做别人做不到的事。首先编写一个待追踪的示例，这个例子很简单，接收用户的输入内容，并原样输出即可，如果输入中包含end，那么程序自动结束，如下
```java
import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.IOException;
import java.io.InputStreamReader;

public class BytemanDemo {
    public static void main(String[] args) {
        new BytemanDemo().start();
    }

    private void start() {
        new Thread(() -> {
            DataInputStream in = new DataInputStream(System.in);
            BufferedReader buf = new BufferedReader(new InputStreamReader(in));
            try {
                String next = buf.readLine();
                while (next != null && !next.contains("end")) {
                    consume(next);
                    next = buf.readLine();
                }

            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    public void consume(String text) {
        //这是个局部变量，将会在byteman中追踪她
        final String arg = text;
        Thread thread = new Thread(() -> System.out.println("program confirm " + arg));
        thread.start();
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return;
    }
}
```

好了，启动程序测试一下
>*byteman test demo*
program confirm byteman test demo

那么试想一下，如果程序输出的和我们输入的总是不同，会不会是consume方法中出现了Bug呢？那么如果想要确认传入到consume方法中的参数是否正确要怎么做呢？所以这时候就需要在byteman中读取consume方法中的局部变量`arg`了。

首先加载byteman到JVM中，并将它attach到需要监听的进程上，指定byteman监控程序监听55000端口（这个端口是用来对byteman的脚本进行响应的）
```cmd
C:\Users\wangxu>jps
18592 Jps
20068 AppMain
8584
18668 Launcher

C:\Users\wangxu>bminstall -b -Dorg.jboss.byteman.transform.all -Dorg.jboss.byteman.verbose -p 55000 20068
```
再回到被监控程序的控制台看到如下输出
>*byteman test demo*
program confirm byteman test demo
Setting org.jboss.byteman.transform.all=
Setting org.jboss.byteman.verbose=
TransformListener() : accepting requests on localhost:55000

从输出信息中可以看到，byteman监听55000端口，并在该端口接收请求，那就来请求一把呗，打开编辑器，编写byteman的btm脚本，ShowLocalVar.btm，如下
```shell
RULE trace line local var

CLASS BytemanDemo
METHOD consume(String)
AFTER WRITE $arg

IF TRUE
        DO traceln("*** transfer value is : " + $arg + " ***")
ENDRULE
```

接下来就安装脚本吧，运行bmsubmit提交脚本
>C:\Users\wangxu> bmsubmit -p 55000 -l ShowLocalVar.btm
install rule trace line local var

再回到被监控程序的控制台，输入一段测试语句"byteman test demo2"查看输出
```shell
TransformListener() : handling connection on port 55000
retransforming BytemanDemo
org.jboss.byteman.agent.Transformer : possible trigger for rule trace line local var in class BytemanDemo
RuleTriggerMethodAdapter.injectTriggerPoint : inserting trigger into BytemanDemo.consume(java.lang.String) void for rule trace line local var
org.jboss.byteman.agent.Transformer : inserted trigger for trace line local var in class BytemanDemo
byteman test demo2
Rule.execute called for trace line local var_0
HelperManager.install for helper class org.jboss.byteman.rule.helper.Helper
calling activated() for helper class org.jboss.byteman.rule.helper.Helper
Default helper activated
calling installed(trace line local var) for helper classorg.jboss.byteman.rule.helper.Helper
Installed rule using default helper : trace line local var
trace line local var execute
*** transfer value is : byteman test demo2 ***
program confirm byteman test demo2
```

可以看到星号打头和结尾的那一句，就是监控程序从被监控程序中得到的局部变量，是不是很强大啊。

当然了可以安装脚本，就可以卸载脚本，同样是使用bmsubmit命令
>C:\Users\wangxu> bmsubmit -p 55000 -u ShowLocalVar.btm
uninstall RULE trace line local var

卸载完成之后，再回到被监控程序的控制台，输入测试语句
```cmd
TransformListener() : handling connection on port 55000
retransforming BytemanDemo
HelperManager.uninstall for helper class org.jboss.byteman.rule.helper.Helper
calling uninstalled(trace line local var) for helper class org.jboss.byteman.rule.helper.Helper
Uninstalled rule using default helper : trace line local var
calling deactivated() for helper classorg.jboss.byteman.rule.helper.Helper
Default helper deactivated
byteman test demo3
program confirm byteman test demo3
```

可以看到程序又回到了最初的摸样，这种完全无侵入式的设计真是太完美了，对于需要被监控的程序来说几乎是透明的，它不需要做任何改变，我们就可以通过动态追踪技术得到我们感兴趣的信息，而且在追踪结束后卸载脚本，被监控程序依然还是最初的摸样。

注意点：我的被监控程序是在IDEA中运行的，也就是说，IDEA会自动帮我编译源码，我只需要运行即可，那么如果你是使用命令行自己编译的Java源码，很可能无法追踪到局部变量的值，这是因为自己编译的话，需要指定`-g`参数，也就是像这样`javac -g Byteman.java`编译出来的class文件才能够被追踪到局部变量。对细节感兴趣的话请看[这里](https://developer.jboss.org/thread/176913)。

### 使用示例（统计方法耗时）
与上例不同，这个例子主要展示如何使用`bmjava`和`bmcheck`这两个命令。先上待追踪代码，很简单的例子，working方法模拟工作方法，我们通过动态追踪获得工作方法花费的时间

```java
import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.concurrent.TimeUnit;

public class WorkerDemo {
    public static void main(String[] args) {
        WorkerDemo workerDemo = new WorkerDemo();
        DataInputStream dataInputStream = new DataInputStream(System.in);
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(dataInputStream));
        try {
            String next;
            while ((next = bufferedReader.readLine()) != null) {
                System.out.println("before working");
                workerDemo.working();
                System.out.println("after working");
                if (next.contains("end")) {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void working() {
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

编写byteman脚本
```shell
RULE trace worker time start

CLASS WorkerDemo
METHOD working
AT ENTRY

IF TRUE
DO createTimer($0)
ENDRULE

RULE trace worker time stop

CLASS WorkerDemo
METHOD working
AT EXIT

IF TRUE
DO traceln("*** working time " + getElapsedTimeFromTimer($0) + " ms ***");
    deleteTimer($0);
ENDRULE
```
注意一点，在规则脚本中用到的方法`createTimer`，`getElapsedTimeFromTimer`，`deleteTimer`都是在`org.jboss.byteman.rule.helper.Helper`类中定义好的，并且在创建Timer之后，一定要记得删除，这里面方法的形参必须要从0开始，感兴趣的可以看看这几个方法的详细说明

- **createTimer** can be called to create a new Timer associated with o. createTimer returns true if a new
Timer was created and false if a Timer associated with o already exists.
- **getElapsedTimeFromTimer** can be called to obtain the number of elapsed milliseconds since the Timer
associated with o was created or since the last call to resetTimer. If no timer associated with o exists a
new timer is created before returning the elapsed time.
- **resetTimer** can be called to zero the Timer associated with o. It returns the number of seconds since the
Timer was created or since the last previous call to resetTimer If no timer associated with o exists a
new timer is created before returning the elapsed time.
- **deleteTimer** can be called to delete the Timer associated with o. deleteTimer returns true if a new Timer
was deleted and false if no Timer associated with o exists.

将WorkerDemo类和规则脚本置于同一目录下，打开cmd运行命令行，先检查一下规则脚本是否合法

```
C:\Users\wangxu>bmcheck -cp . -v TimeTracer.btm
Checking rule trace worker time start against class WorkerDemo
Parsed rule "trace worker time start" for class WorkerDemo
Type checked rule "trace worker time start"

Checking rule trace worker time stop against class WorkerDemo
Parsed rule "trace worker time stop" for class WorkerDemo
Type checked rule "trace worker time stop"

TestScript: no errors
```
`bmcheck`命令可以在应用规则脚本之前对其进行检查，`bmcheck [-cp classpath]* [-p package]* [-v] script1 . . . scriptN`，如果是带包名的类，那么需要使用`-p`参数即可。

再将脚本和程序一起启动
```
C:\Users\wangxu>javac -g WorkerDemo.java

C:\Users\wangxu>bmjava -l TimeTracer.btm WorkerDemo
trace working time demo
before working
*** working time 5003 ms ***
after working
trace end
before working
*** working time 5000 ms ***
after working
```

这样操作有一个限制，就是无法追踪已经启动了的程序，如果已经有个程序正在运行，而这时候程序的输出与我们预期的不符，如何用脚本来追踪呢？这时候需要用到`bminstall`和`bmsubmit`命令。首先就按最普通的方式启动程序再说
>C:\Users\wangxu>java WorkerDemo

再开一个cmd命令行，先把byteman attach到需要追踪的进程上，再提交规则脚本
```
C:\Users\wangxu>jps
11724
1468 Jps
17308 WorkerDemo
5948 Launcher
C:\Users\wangxu>bminstall -b -Dorg.jboss.byteman.tranform.all 17308

C:\Users\wangxu>bminstall -b -Dorg.jboss.byteman.tranform.all 17308

C:\Users\wangxu>bmsubmit -l TimeTracer.btm
install rule trace worker time start
install rule trace worker time stop
```
这时候再回到启动程序的命令行，可以看到有新的输出
>Setting org.jboss.byteman.tranform.all=

我们输入一行测试语句试一下，看看规则脚本是否生效
```
C:\Users\wangxu>java WorkerDemo
Setting org.jboss.byteman.tranform.all=
trace working time demo2 and then end
before working
*** working time 5007 ms ***
after working
```
使用这种方式就可以动态的追踪已经启动的程序了。

### Byteman脚本介绍
首先了解byteman的脚本结构，如下所示
```shell
######################################
# Example Rule Set
#
# a single rule definition
RULE example rule
# comment line in rule body
. . .
ENDRULE
```

其中合法的规则事件可以是如下这些
```shell
# rule skeleton
RULE <rule name>
CLASS <class name>
METHOD <method name>
BIND <bindings>
IF  <condition>
DO  <actions>
ENDRULE
```

同样全部的合法定位点或者说注入点如下所示
```shell
AT ENTRY
AT EXIT
AT LINE number
AT READ [type .] field [count | ALL ]
AT READ $var-or-idx [count | ALL ]
AFTER READ [ type .] field [count | ALL ]
AFTER READ $var-or-idx [count | ALL ]
AT WRITE [ type .] field [count | ALL ]
AT WRITE $var-or-idx [count | ALL ]
AFTER WRITE [ type .] field [count | ALL ]
AFTER WRITE $var-or-idx [count | ALL ]
AT INVOKE [ type .] method [ ( argtypes ) ] [count | ALL ]
AFTER INVOKE [ type .] method [ ( argtypes ) ][count | ALL ]
AT SYNCHRONIZE [count | ALL ]
AFTER SYNCHRONIZE [count | ALL ]
AT THROW [count | ALL ]
AT EXCEPTION EXIT
```

更多更详细的信息可以去参考Byteman的官方手册。也许你注意到了，在脚本中我们使用了traceln语句，那么这个调用的其实是Byteman的`org.jboss.byteman.rule.helper.Helper`类的方法，这些方法都是已经内置的，可以直接在脚本中调用。

当然了如果你觉得这些内置的方法不够用，也可以使用自定义的Helper类，例如官方给出的一个例子是
```shell
# helper example 2
RULE help yourself but rely on others
CLASS com.arjuna.wst11.messaging.engines.CoordinatorEngine
METHOD commit
HELPER HelperSub
AT ENTRY
IF NOT flagged($this)
DO debug("throwing wrong state");
 flag($this);
 throw new WrongStateException()
ENDRULE
```

HelperSub的源码示例如下
```java
class HelperSub extends Helper {
  public boolean debug(String message) {
      super("!!! IMPORTANT EVENT !!! " + message);
  }
}
```

### Byteman环境变量
在Byteman中，总共的环境变量有如下这些
* org.jboss.byteman.compileToBytecode
* org.jboss.byteman.dump.generated.classes
* org.jboss.byteman.dump.generated.classes.directory
* org.jboss.byteman.dump.generated.classes.intermediate
* org.jboss.byteman.verbose
* org.jboss.byteman.debug
* org.jboss.byteman.transform.all
* org.jboss.byteman.skip.overriding.rules
* org.jboss.byteman.allow.config.updates
* org.jboss.byteman.sysprops.strict

而我们已经使用过了其中的两个，就是在bminstall命令中所示的
* org.jboss.byteman.transform.all 如果设置了将允许注入java.lang和其子包的class
* org.jboss.byteman.verbose 如果设置将显示执行的各种跟踪信息到System.out，包括类型检查，编译，和执行规则

有关更加详细的解释可以到[官方网站](http://downloads.jboss.org/byteman/3.0.10/ProgrammersGuide.html#environment-settings)中的**Environment Settings**一节查看。

### javaagent技术
简单来说，javaagent 技术是一个开发者可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在 JVM 上的程序，甚至能够替换和修改某些类的定义。有了这样的功能，开发者就可以实现更为灵活的运行时虚拟机监控和 Java 类操作了，这样的特性实际上提供了一种虚拟机级别支持的 AOP 实现方式，使得开发者无需对 JDK 做任何升级和改动，就可以实现某些 AOP 的功能了。有关 javaagent 更详细的介绍，可以参看[Instrumentation 简介](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)这篇文章。

#### 用javaagent加载规则文件并启动程序
javaagent 选项支持在所有的个人机或服务器的JVM中使用Byteman，那么看一下如何使用 javaagent 技术来启动Byteman吧。还是以BytemanDemo为例，BytemanDemo和ShowLocalVar.btm文件都在`C:/Users/wangxu`路径下
```
C:\Users\wangxu>javac -g BytemanDemo.java

C:\Users\wangxu>java -javaagent:%BYTEMAN_HOME%\lib\byteman.jar=script:ShowLocalVar.btm BytemanDemo
javaagent byteman demo
*** transfer value is : javaagent byteman demo ***
program confirm javaagent byteman demo
```

`script`用于指示 Byteman 规则文件的位置。Byteman agent 读取到这个选项之后从规则文件中加载和注入Byteman规则。如果要加载多个`script:file`规则文件，使用逗号（,）分隔即可。

#### 用javaagent启动程序并动态加载规则文件
这种方式是将规则文件和需要监控的程序一起启动，那么如果被监控程序已经启动，该如何注入规则文件呢？严格执行如下步骤即可
1. 需要在启动程序的时候采用`java -javaagent`方式启动，同样地先编译`javac -g BytemanDemo.java`
2. 新开cmd，输入
```shell
C:\Users\wangxu>java -javaagent:%BYTEMAN_HOME%\lib\byteman.jar=listener:true,boot:%BYTEMAN_HOME%\lib\byteman.jar -Dorg.jboss.byteman.transform.all BytemanDemo
```
  Linux系统只需要把`%BYTEMAN_HOME%`换成`${BYTEMAN_HOME}`即可，注意Linux上分隔符是正斜杠，这个监听器会开启一个网络监听。注意：当没有规则加载的时候，程序的的行为不会发生任何变化，仅输出我们输入的内容。这时候需要第三步来加载规则文件
3. 新开cmd，输入`C:\Users\wangxu>bmsubmit -l ShowLocalVar.btm`，cmd反馈出`install rule trace line local var`，再回到第二步的cmd窗口，输入测试语句"javaagent byteman demo2"
```shell
C:\Users\wangxu>java -javaagent:%BYTEMAN_HOME%\lib\byteman.jar=listener:true,boot:%BYTEMAN_HOME%\lib\byteman.jar -Dorg.jboss.byteman.transform.all BytemanDemo
javaagent byteman demo2
*** transfer value is : javaagent byteman demo2 ***
program confirm javaagent byteman demo2
```
  可以看到规则已经生效。这种方式虽然可以实现同样地监控，但是还是存在一个明显的弊端，就是要求在启动程序的时候使用`javaagent`方式，但是通常我们并不会用这种方式，大多数人都是用最普通的`java -jar`命令，那么且看下节。

#### 动态安装agent到正在运行的程序中
如果你启动了一个长时程序，并且没有加载 Byteman agent。你不需要重启程序也可以使用 Byteman。在Byteman的bin目录下有很多脚本，Windows上的是`bminstall.bat`，Linux上的是`bminstall.sh`，由于所有线上服务器都是用的Linux，那么接下来我就在CentOS上演示一下动态安装agent技术。

首先编译，启动程序再说
```shell
[elastic@escluster ~]$ ll
total 12
-rw-rw-r--.  1 elastic elastic 1218 Sep 21 15:59 BytemanDemo.java
drwxr-xr-x. 10 elastic elastic 4096 Aug 29 14:45 elasticsearch
-rw-rw-r--.  1 elastic elastic  171 Sep 21 15:59 ShowLocalVar.btm
[elastic@escluster ~]$ javac -g BytemanDemo.java
[elastic@escluster ~]$ java BytemanDemo
Byteman agent in linux
program confirm Byteman agent in linux
```

这个时候可见就是一个普通的程序，下面在Linux配置环境变量，编辑`/etc/profile`
```shell
BYTEMAN_HOME=/usr/program/byteman-3.0.10
PATH=$PATH:$BYTEMAN_HOME/bin
export BYTEMAN_HOME
export PATH
```

新开一个终端，按如下所示操作
```shell
[elastic@escluster ~]$ jps
24739 Elasticsearch
12215 BytemanDemo
25128 Elasticsearch
24874 Elasticsearch
12251 Jps
[elastic@escluster ~]$ bminstall.sh -b -Dorg.jboss.byteman.transform.all 12215
[elastic@escluster ~]$ bmsubmit.sh -l ShowLocalVar.btm
install rule trace line local var
```

`bminstall.sh`并没有加载任何规则脚本，只是开启了agent监听器。然后就可以通过bmsubmit.sh提交规则。在将规则脚本sumbit或者unsubmit到程序中，会看到程序的行为被动态修改了。返回到第一个终端，继续输入测试语句
```
[elastic@escluster ~]$ java BytemanDemo
Byteman agent in linux
program confirm Byteman agent in linux
Setting org.jboss.byteman.transform.all=
Byteman agent in linux after install and submit
*** transfer value is : Byteman agent in linux after install and submit ***
program confirm Byteman agent in linux after install and submit
```

Perfect规则生效了，无论是Windows还是Linux，Byteman都可以完美地工作。

### Byteman其它命令
Byteman的启动方式非常多，个人精力有限，我不能一一介绍，想要了解更多的信息，直接去翻阅压缩包中的`docs/byteman-programmers-guide.pdf`用户指南，上面有详细地介绍。

在解压的Byteman的bin目录下，有非常多的脚本，简单介绍如下所示
- **bmcheck** 在注入规则文件之前，该命令可以在线下对你的规则脚本进行解析和类型检查
- **bminstall** 安装agent到一个正在运行的程序中
- **bmjava** 该脚本包装了`-javaagent`选项。它的用法很像java命令，但是它能以`-javaagent script:`选项的方式接受Byteman规则脚本。并且自动以`boot:`的方式绑定了Byteman的Jar文件
- **bmsetenv** 该脚本用来设置环境，agent对配置其行为的各种环境设置非常敏感
- **bmsubmit** 上传和卸载规则脚本





