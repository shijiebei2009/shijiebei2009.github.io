title: Log4j2实现不同线程不同级别日志输出到不同的文件中
date: 2016-12-18 20:00:16
tags: [Log4j2]
categories: Programming Notes

---

### Java日志框架
作为一个Java程序员，肯定离不开日志框架，现在最优秀的Java日志框架是Log4j2，没有之一。根据官方的测试表明，在多线程环境下，Log4j2的异步日志表现更加优秀。在异步日志中，Log4j2使用独立的线程去执行I/O操作，可以极大地提升应用程序的性能。

在官方的测试中，下图比较了`Sync`、`Async Appenders`和`Loggers all async`三者的性能。其中`Loggers all async`表现最为出色，而且线程数越多，`Loggers all async`性能越好。

![](http://7xig3q.com1.z0.glb.clouddn.com/log4j2-async-vs-sync-throughput.png)

除了对Log4j2自身的不同模式做对比以外，官方还做了**Log4j2/Log4j1/Logback**的对比，如下图所示

![](http://7xig3q.com1.z0.glb.clouddn.com/log4j-async-throughput-comparison.png)

其中，`Loggers all async`是基于[LMAX Disruptor](http://lmax-exchange.github.com/disruptor/)实现的。

### 使用Log4j2

#### 需要哪些JAR
使用Log4j2最少需要两个JAR，分别是`log4j-api-2.x`和`log4j-core-2.x`，其它JAR包根据应用程序需要添加。

#### 配置文件位置
默认的，Log4j2在**classpath**下寻找名为`log4j2.xml`的配置文件。也可以使用`system property`指定配置文件的全路径。`-Dlog4j.configurationFile=path/to/log4j2.xml`，在Java代码中指定路径如下所示
```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.core.LoggerContext;

import java.io.File;

public class Demo {
    public static void main(String[] args) {
        LoggerContext loggerContext = (LoggerContext) LogManager.getContext(false);
        File file = new File("path/to/a/different/log4j2.xml");
        loggerContext.setConfigLocation(file.toURI());
    }
}
```

一般的，不需要手动关闭Log4j2，如果想手动在代码中关闭Log4j2如下所示

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.core.LoggerContext;
import org.apache.logging.log4j.core.config.Configurator;

public class Demo {
    public static void main(String[] args) {
        Configurator.shutdown((LoggerContext) LogManager.getContext());
    }
}
```

有关Log4j2的内容很多，不能一一列出，如果在开发中遇到任何问题，推荐去[官方文档](http://logging.apache.org/log4j/2.x/)中寻找解决方案。

#### 不同的线程输出日志到不同的文件中
##### 方法一使用ThreadContext
在多线程编程中，如果不做特殊的设置，那么多个线程的日志会输出到同一个日志文件中，这样在查阅日志的时候，会带来诸多不便。很自然地，我们想到了让不同的线程输出日志到不同的文件中，这样不是更好吗？在翻阅官方文档过程中，找到了**FAQ**（Frequently Asked Questions），其中有个问题**[How do I dynamically write to separate log files?](http://logging.apache.org/log4j/2.x/faq.html#separate_log_files)**正是我们所需要的。根据提示步步推进可以顺利解决该问题。其中**log4j2.xml**配置如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="OFF">
    <Appenders>
        <Routing name="Routing">
            <Routes pattern="$${ctx:ROUTINGKEY}">
                <!-- This route is chosen if ThreadContext has value 'special' for key ROUTINGKEY. -->
                <Route key="special">
                    <RollingFile name="Rolling-${ctx:ROUTINGKEY}" fileName="logs/special-${ctx:ROUTINGKEY}.log"
                                 filePattern="./logs/${date:yyyy-MM}/${ctx:ROUTINGKEY}-special-%d{yyyy-MM-dd}-%i.log.gz">
                        <PatternLayout>
                            <Pattern>%d{ISO8601} [%t] %p %c{3} - %m%n</Pattern>
                        </PatternLayout>
                        <Policies>
                            <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
                            <SizeBasedTriggeringPolicy size="10 MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
                <!-- This route is chosen if ThreadContext has no value for key ROUTINGKEY. -->
                <Route key="$${ctx:ROUTINGKEY}">
                    <RollingFile name="Rolling-default" fileName="logs/default.log"
                                 filePattern="./logs/${date:yyyy-MM}/default-%d{yyyy-MM-dd}-%i.log.gz">
                        <PatternLayout>
                            <pattern>%d{ISO8601} [%t] %p %c{3} - %m%n</pattern>
                        </PatternLayout>
                        <Policies>
                            <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
                            <SizeBasedTriggeringPolicy size="10 MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
                <!-- This route is chosen if ThreadContext has a value for ROUTINGKEY
                     (other than the value 'special' which had its own route above).
                     The value dynamically determines the name of the log file. -->
                <Route>
                    <RollingFile name="Rolling-${ctx:ROUTINGKEY}" fileName="logs/other-${ctx:ROUTINGKEY}.log"
                                 filePattern="./logs/${date:yyyy-MM}/${ctx:ROUTINGKEY}-other-%d{yyyy-MM-dd}-%i.log.gz">
                        <PatternLayout>
                            <pattern>%d{ISO8601} [%t] %p %c{3} - %m%n</pattern>
                        </PatternLayout>
                        <Policies>
                            <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
                            <SizeBasedTriggeringPolicy size="10 MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
            </Routes>
        </Routing>
        <!--很直白，Console指定了结果输出到控制台-->
        <Console name="ConsolePrint" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %t %-5level %class{36} %L %M - %msg%xEx%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <!-- 级别顺序（低到高）：TRACE < DEBUG < INFO < WARN < ERROR < FATAL -->
        <Root level="DEBUG" includeLocation="true">
            <!--AppenderRef中的ref值必须是在前面定义的appenders-->
            <AppenderRef ref="Routing"/>
            <AppenderRef ref="ConsolePrint"/>
        </Root>
    </Loggers>
</Configuration>
```

测试类如下所示
```java
import lombok.extern.log4j.Log4j2;
import org.apache.logging.log4j.ThreadContext;

@Log4j2
public class TestLog {
    public static void main(String[] args) {
        new Thread(() -> {
            ThreadContext.put("ROUTINGKEY", Thread.currentThread().getName());
            log.info("info");
            log.debug("debug");
            log.error("error");
            ThreadContext.remove("ROUTINGKEY");
        }).start();
        new Thread(() -> {
            ThreadContext.put("ROUTINGKEY", Thread.currentThread().getName());
            log.info("info");
            log.debug("debug");
            log.error("error");
            ThreadContext.remove("ROUTINGKEY");
        }).start();
    }
}
```
运行测试类，会得到如下两个日志文件，`other-Thread-1.log`和`other-Thread-2.log`，每个日志文件对应着一个线程。该程序使用Gradle构建，依赖的JAR包如下：
```Groovy
dependencies {
    compile 'org.projectlombok:lombok:1.16.10'
    compile 'org.apache.logging.log4j:log4j-core:2.6'
    compile 'org.apache.logging.log4j:log4j-api:2.6'
}
```

需要注意的一点是，每次在使用**log**对象之前，需要先设置`ThreadContext.put("ROUTINGKEY", Thread.currentThread().getName());`，设置的**key**和`log4j2.xml`配置文件中的**key**要一致，而**value**可以是任意值，参考配置文件即可理解。

有没有发现，每次使用log对象，还需要添加额外的代码，这不是恶心他妈给恶心开门——恶心到家了吗？有没有更加优雅地解决办法呢？且看下节。

##### 方法二实现StrLookup
修改`log4j2.xml`配置文件如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="OFF">
    <Appenders>
        <Routing name="Routing">
            <Routes pattern="$${thread:threadName}">
                <Route>
                    <RollingFile name="logFile-${thread:threadName}"
                                 fileName="logs/concurrent-${thread:threadName}.log"
                                 filePattern="logs/concurrent-${thread:threadName}-%d{MM-dd-yyyy}-%i.log">
                        <PatternLayout pattern="%d %-5p [%t] %C{2} - %m%n"/>
                        <Policies>
                            <SizeBasedTriggeringPolicy size="50 MB"/>
                        </Policies>
                        <DefaultRolloverStrategy max="100"/>
                    </RollingFile>
                </Route>
            </Routes>
        </Routing>
        <Async name="async" bufferSize="1000" includeLocation="true">
            <AppenderRef ref="Routing"/>
        </Async>
        <!--很直白，Console指定了结果输出到控制台-->
        <Console name="ConsolePrint" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %t %-5level %class{36} %L %M - %msg%xEx%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info" includeLocation="true">
            <AppenderRef ref="async"/>
            <AppenderRef ref="ConsolePrint"/>
        </Root>
    </Loggers>
</Configuration>
```

实现StrLookup中的lookup方法，代码如下：
```java
import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.config.plugins.Plugin;
import org.apache.logging.log4j.core.lookup.StrLookup;

@Plugin(name = "thread", category = StrLookup.CATEGORY)
public class ThreadLookup implements StrLookup {
    @Override
    public String lookup(String key) {
        return Thread.currentThread().getName();
    }

    @Override
    public String lookup(LogEvent event, String key) {
        return event.getThreadName() == null ? Thread.currentThread().getName()
                : event.getThreadName();
    }

}
```

编写测试类，代码如下：
```java
import lombok.extern.log4j.Log4j2;

@Log4j2
public class TestLog {
    public static void main(String[] args) {
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
    }
}
```
该测试类同样会得到两个日志文件，每个线程分别对应一个，但是在使用**log**对象之前不再需要设置`ThreadContext.put("ROUTINGKEY", Thread.currentThread().getName());`，是不是清爽多了。

根据官方的性能测试我们知道，`Loggers all async`的性能最高，但是方法一使用的是`Sync`模式（因为Appender默认是synchronous的），方法二使用的是`Async Appender`模式，那么如何更进一步让所有的**Loggers**都是**Asynchronous**的，让我们的配置更完美呢？想要使用`Loggers all async`还需要做一步设置，如果是Maven或Gradle项目，需要在`src/main/resources`目录下添加**log4j2.component.properties**配置文件，根据[官网说明](http://logging.apache.org/log4j/2.x/manual/async.html)，其中内容为

```
Log4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

同时还需要在**classpath**下添加依赖的**disruptor** JAR包`com.lmax:disruptor:3.3.6`。当配置使用了`AsyncLoggerContextSelector`之后，所有的**Loggers**就都是异步的了。有方法证明使用了`Loggers all async`吗，答案是有，默认的**location**不会传递给`Loggers all async`的I/O线程，所以如果不设置`includeLocation=true`的话，打印出来的日志中**location**信息是“?”，例如“2016-12-16 16:38:47,285 INFO  [Thread-3] ? - info”，如果设置includeLocation="true"之后，打印出“2016-12-16 16:39:14,899 INFO  [Thread-3] TestLog - info”，Gradle构建依赖如下：
```Groovy
dependencies {
    compile 'org.projectlombok:lombok:1.16.10'
    compile 'org.apache.logging.log4j:log4j-core:2.6'
    compile 'org.apache.logging.log4j:log4j-api:2.6'
    compile 'com.lmax:disruptor:3.3.6'
}
```

#### 不同级别的日志输出到不同的文件中
通常情况下，用到的日志级别主要是info/debug/error三个，而如果不做特殊配置，这三者信息是写到一个日志文件中的，当我们需要不同级别的日志输出到不同的文件中时，需要如何做呢？`log4j2.xml`配置信息如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="OFF">
    <Properties>
        <Property name="logFilePath">logs</Property>
        <Property name="logFileName">testLog</Property>
    </Properties>
    <Appenders>
        <!--很直白，Console指定了结果输出到控制台-->
        <Console name="ConsolePrint" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %t %-5level %class{36} %L %M - %msg%xEx%n"/>
        </Console>
        <!--<File>输出结果到指定文件</File>-->
        <!--<RollingFile>同样输出结果到指定文件，但是使用buffer，速度会快点</RollingFile>-->
        <!--filePattern：表示当日志到达指定的大小或者时间，产生新日志时，旧日志的命名路径。-->
        <!--PatternLayout：和log4j一样，指定输出日志的格式，append表示是否追加内容，值默认为true-->
        <RollingFile name="RollingFileDebug" fileName="${logFilePath}/${logFileName}-debug.log"
                     filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}_%i.log.gz">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
            <!--注意，如果有多个ThresholdFilter，那么Filters标签是必须的-->
            <Filters>
                <!--首先需要过滤不符合的日志级别，把不需要的首先DENY掉，然后在ACCEPT需要的日志级别，次序不能颠倒-->
                <!--INFO及以上级别拒绝输出-->
                <ThresholdFilter level="INFO" onMatch="DENY" onMismatch="NEUTRAL"/>
                <!--只输出DEBUG级别信息-->
                <ThresholdFilter level="DEBUG" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <Policies>
                <!--时间策略，每隔24小时产生新的日志文件-->
                <TimeBasedTriggeringPolicy/>
                <!--大小策略，每到30M时产生新的日志文件-->
                <SizeBasedTriggeringPolicy size="30MB"/>
            </Policies>
        </RollingFile>

        <RollingFile name="RollingFileInfo" fileName="${logFilePath}/${logFileName}-info.log"
                     filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}_%i.log.gz">
            <Filters>
                <!--onMatch:Action to take when the filter matches. The default value is NEUTRAL-->
                <!--onMismatch:    Action to take when the filter does not match. The default value is DENY-->
                <!--级别在ERROR之上的都拒绝输出-->
                <!--在组合过滤器中，接受使用NEUTRAL（中立），被第一个过滤器接受的日志信息，会继续用后面的过滤器进行过滤，只有符合所有过滤器条件的日志信息，才会被最终写入日志文件-->
                <ThresholdFilter level="ERROR" onMatch="DENY" onMismatch="NEUTRAL"/>
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="30MB"/>
            </Policies>
        </RollingFile>

        <RollingFile name="RollingFileError" fileName="${logFilePath}/${logFileName}-error.log"
                     filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}_%i.log.gz">
            <Filters>
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="30MB"/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <!--logger用于定义log的level以及所采用的appender，如果无需自定义，可以使用root解决，root标签是log的默认输出形式-->
        <!-- 级别顺序（低到高）：TRACE < DEBUG < INFO < WARN < ERROR < FATAL -->
        <Root level="DEBUG" includeLocation="true">
            <!-- 只要是级别比ERROR高的，包括ERROR就输出到控制台 -->
            <!--appender-ref中的值必须是在前面定义的appenders-->
            <Appender-ref level="ERROR" ref="ConsolePrint"/>
            <Appender-ref ref="RollingFileDebug"/>
            <Appender-ref ref="RollingFileInfo"/>
            <Appender-ref ref="RollingFileError"/>
        </Root>
    </Loggers>
</Configuration>
```

`src\main\resources\log4j2.component.properties`内容不变，如下
```
Log4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

测试代码如下

```java
import lombok.extern.log4j.Log4j2;

@Log4j2
public class TestLog {
    public static void main(String[] args) {
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
    }
}
```

该程序会生成三份日志文件【testLog-debug.log，testLog-error.log，testLog-info.log】，如果你足够细心的话，就会发现线程1和线程2的**info|debug|error**信息都输出到一个**info|debug|error**日志文件中了。如何解决这个问题呢？换句话说，我想得到
>Thread 1
>>Thread 1的info日志
  Thread 1的debug日志
  Thread 1的error日志

>Thread 2
>>Thread 2的info日志
  Thread 2的debug日志
  Thread 2的error日志

六个日志文件，要如何实现呢？这正是下一节要讲述的内容。

#### 不同线程不同级别的日志输出到不同的文件中
要实现该功能，还要从**RoutingAppender**身上做文章。**RoutingAppender**主要用来评估LogEvents，然后将它们路由到下级Appender。目标Appender可以是先前配置的并且可以由其名称引用的Appender，或者可以根据需要动态地创建Appender。**RoutingAppender**应该在其引用的任何Appenders之后配置，以确保它可以正确关闭。

**RoutingAppender**中的name属性用来指定该Appender的名字，它可以包含多个Routes子节点，用来标识选择Appender的条件，而Routes只有一个属性“pattern”，该pattern用来评估所有注册的**Lookups**，并且其结果用于选择路由。在Routes下可以有多个Route，每个Route都必须配置一个key，如果这个key匹配“pattern”的评估结果，那么这个Route就被选中。同时每个Route都必须引用一个Appender，如果这个Route包含一个ref属性，那么这个Route将引用一个在配置中定义的Appender，如果这个Route包含一个Appender的定义，那么这个Appender将会根据RoutingAppender的上下文创建并被重用。

废话说多了，直接上配置才简洁明了。log4j2.xml配置如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--status的含义为是否记录log4j2本身的event信息，默认是OFF-->
<Configuration status="OFF">
    <Properties>
        <!--自定义一些常量，之后使用${变量名}引用-->
        <Property name="logFilePath">logs</Property>
        <Property name="logFileName">testLog</Property>
    </Properties>
    <Appenders>
        <!--很直白，Console指定了结果输出到控制台-->
        <Console name="ConsolePrint" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %t %-5level %class{36} %L %M - %msg%xEx%n"/>
        </Console>
        <!--<File>输出结果到指定文件</File>-->
        <!--<RollingFile>同样输出结果到指定文件，但是使用buffer，速度会快点</RollingFile>-->
        <!--filePattern：表示当日志到达指定的大小或者时间，产生新日志时，旧日志的命名路径。-->
        <!--PatternLayout：和log4j一样，指定输出日志的格式，append表示是否追加内容，值默认为true-->
        <Routing name="RollingFileDebug_${thread:threadName}">
            <Routes pattern="$${thread:threadName}">
                <Route>
                    <RollingFile name="RollingFileDebug_${thread:threadName}"
                                 fileName="${logFilePath}/${logFileName}_${thread:threadName}_debug.log"
                                 filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}-${thread:threadName}-debug_%i.log.gz">
                        <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
                        <!--注意，如果有多个ThresholdFilter，那么Filters标签是必须的-->
                        <Filters>
                            <!--首先需要过滤不符合的日志级别，把不需要的首先DENY掉，然后在ACCEPT需要的日志级别，次序不能颠倒-->
                            <!--INFO及以上级别拒绝输出-->
                            <ThresholdFilter level="INFO" onMatch="DENY" onMismatch="NEUTRAL"/>
                            <!--只输出DEBUG级别信息-->
                            <ThresholdFilter level="DEBUG" onMatch="ACCEPT" onMismatch="DENY"/>
                        </Filters>
                        <Policies>
                            <!--时间策略，每隔24小时产生新的日志文件-->
                            <TimeBasedTriggeringPolicy/>
                            <!--大小策略，每到30M时产生新的日志文件-->
                            <SizeBasedTriggeringPolicy size="30MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
            </Routes>
        </Routing>

        <Routing name="RollingFileInfo_${thread:threadName}">
            <Routes pattern="$${thread:threadName}">
                <Route>
                    <RollingFile name="RollingFileInfo_${thread:threadName}"
                                 fileName="${logFilePath}/${logFileName}_${thread:threadName}_info.log"
                                 filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}-${thread:threadName}-info_%i.log.gz">
                        <Filters>
                            <!--onMatch: Action to take when the filter matches. The default value is NEUTRAL-->
                            <!--onMismatch:    Action to take when the filter does not match. The default value is DENY-->
                            <!--级别在ERROR之上的都拒绝输出-->
                            <!--在组合过滤器中，接受使用NEUTRAL（中立），被第一个过滤器接受的日志信息，会继续用后面的过滤器进行过滤，只有符合所有过滤器条件的日志信息，才会被最终写入日志文件-->
                            <ThresholdFilter level="ERROR" onMatch="DENY" onMismatch="NEUTRAL"/>
                            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
                        </Filters>
                        <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
                        <Policies>
                            <TimeBasedTriggeringPolicy/>
                            <SizeBasedTriggeringPolicy size="30MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
            </Routes>
        </Routing>

        <Routing name="RollingFileError_${thread:threadName}">
            <Routes pattern="$${thread:threadName}">
                <Route>
                    <RollingFile name="RollingFileError_${thread:threadName}"
                                 fileName="${logFilePath}/${logFileName}_${thread:threadName}_error.log"
                                 filePattern="${logFilePath}/$${date:yyyy-MM}/${logFileName}-%d{yyyy-MM-dd}-${thread:threadName}-error_%i.log.gz">
                        <Filters>
                            <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
                        </Filters>
                        <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
                        <Policies>
                            <TimeBasedTriggeringPolicy/>
                            <SizeBasedTriggeringPolicy size="30MB"/>
                        </Policies>
                    </RollingFile>
                </Route>
            </Routes>
        </Routing>
        <!--bufferSize整数，指定可以排队的events最大数量，如果使用BlockingQueue，这个数字必须是2的幂次-->
        <!--includeLocation默认值是FALSE，如果指定为TRUE，会降低性能，但是推荐设置为TRUE，否则不打印位置行信息-->
        <Async name="async" bufferSize="262144" includeLocation="true">
            <AppenderRef ref="RollingFileDebug_${thread:threadName}"/>
            <AppenderRef ref="RollingFileInfo_${thread:threadName}"/>
            <AppenderRef ref="RollingFileError_${thread:threadName}"/>
            <!-- 只要是级别比ERROR高的，包括ERROR就输出到控制台 -->
            <AppenderRef ref="ConsolePrint" level="ERROR"/>
        </Async>
    </Appenders>
    <Loggers>
        <!--Logger用于定义log的level以及所采用的appender，如果无需自定义，可以使用root解决，root标签是log的默认输出形式-->
        <!-- 级别顺序（低到高）：TRACE < DEBUG < INFO < WARN < ERROR < FATAL -->
        <Root level="DEBUG" includeLocation="true">
            <!--AppenderRef中的ref值必须是在前面定义的appenders-->
            <AppenderRef ref="async"/>
        </Root>
    </Loggers>
</Configuration>
```

`log4j2.component.properties`和`ThreadLookup`类不变，依赖的JAR包和上一节一样。测试类如下

```java
import lombok.extern.log4j.Log4j2;

@Log4j2
public class TestLog {
    public static void main(String[] args) {
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
        new Thread(() -> {
            log.info("info");
            log.debug("debug");
            log.error("error");
        }).start();
    }
}
```
该程序会输出六个日志文件，分别是
>testLog_Thread-2_debug.log
 testLog_Thread-2_error.log
 testLog_Thread-2_info.log
 testLog_Thread-3_debug.log
 testLog_Thread-3_error.log
 testLog_Thread-3_info.log

至此，就实现了不同线程不同级别的日志输出到不同文件中的功能。

### 如何启用All Loggers Asynchronous
为了使得所有的**Loggers**都是异步的，除了添加一个新的配置文件，就是`log4j2.component.properties`外，还有其它方式吗？有的，仅列举如下
- 例如【IntelliJ IDEA】中使用Gradle构建项目，那么可以在Settings | Build, Execution, Deployment | Build Tools | Gradle | Gradle VM options中填入
```
-DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```
- 另一种就是在前面提到的**ThreadLookup**类中，添加静态代码块
```java
static {
    System.setProperty("Log4jContextSelector", "org.apache.logging.log4j.core.async.AsyncLoggerContextSelector");
}
```
根据参考手册，有一点需要注意的就是，要使用`<Root>`或`<Logger>`标签，而不是`<asyncRoot>`和`<asyncLogger>`，原文如下：
>When AsyncLoggerContextSelector is used to make all loggers asynchronous, make sure to use normal `<root>` and `<logger>` elements in the configuration. The AsyncLoggerContextSelector will ensure that all loggers are asynchronous, using a mechanism that is different from what happens when you configure `<asyncRoot>` or `<asyncLogger>`. The latter elements are intended for mixing async with sync loggers.

### 混合使用Synchronous和Asynchronous Loggers
需要`disruptor-3.0.0.jar`或更高版本的jar包，不需要设置系统属性`Log4jContextSelector`，在配置中可以混合使用`Synchronous`和`asynchronous loggers`，使用`<AsyncRoot>`或者`<AsyncLogger>`去指定需要异步的Loggers，`<AsyncLogger>`元素还可以包含`<Root>`和`<Logger>`用于同步的Loggers。注意如果使用的是`<AsyncRoot>`或者`<AsyncLogger>`，那么就无需设置系统属性`Log4jContextSelector`了。

一个混合了同步和异步的Loggers配置如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- No need to set system property "Log4jContextSelector" to any value
     when using <asyncLogger> or <asyncRoot>. -->
<Configuration status="WARN">
  <Appenders>
    <!-- Async Loggers will auto-flush in batches, so switch off immediateFlush. -->
    <RandomAccessFile name="RandomAccessFile" fileName="asyncWithLocation.log"
              immediateFlush="false" append="false">
      <PatternLayout>
        <Pattern>%d %p %class{1.} [%t] %location %m %ex%n</Pattern>
      </PatternLayout>
    </RandomAccessFile>
  </Appenders>
  <Loggers>
    <!-- pattern layout actually uses location, so we need to include it -->
    <AsyncLogger name="com.foo.Bar" level="trace" includeLocation="true">
      <AppenderRef ref="RandomAccessFile"/>
    </AsyncLogger>
    <Root level="info" includeLocation="true">
      <AppenderRef ref="RandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

**参考资料**
【1】[Different log files for multiple threads using log4j2](http://stackoverflow.com/questions/19976211/different-log-files-for-multiple-threads-using-log4j2)
【2】[log4j2: Location for setting Log4jContextSelector system property for asynchronous logging](http://stackoverflow.com/questions/27154558/log4j2-location-for-setting-log4jcontextselector-system-property-for-asynchrono)
【3】[Making All Loggers Asynchronous](https://logging.apache.org/log4j/2.x/manual/async.html#AllAsync)
【4】[AsyncAppender](https://logging.apache.org/log4j/2.x/manual/appenders.html#AsyncAppender)
【5】[Mixing Synchronous and Asynchronous Loggers](https://logging.apache.org/log4j/2.x/manual/async.html#MixedSync-Async)
【6】[How to verify log4j2 is logging asynchronously via LMAX disruptor?](http://stackoverflow.com/questions/33293059/how-to-verify-log4j2-is-logging-asynchronously-via-lmax-disruptor)
【7】[Log4j2 synchronous logging](http://stackoverflow.com/questions/38043488/log4j2-synchronous-logging)
【8】[Log4J2 sync loggers faster than mixed async/sync loggers](http://stackoverflow.com/questions/30338034/log4j2-sync-loggers-faster-than-mixed-async-sync-loggers)
【9】[Difference between Asynclogger and AsyncAppender in Log4j2](http://stackoverflow.com/questions/24177601/difference-between-asynclogger-and-asyncappender-in-log4j2)