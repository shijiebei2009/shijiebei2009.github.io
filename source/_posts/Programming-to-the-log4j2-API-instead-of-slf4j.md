---
title: 面向log4j2 API编程而不是slf4j
date: 2017-03-08 22:18:55
tags: [Log4j2]
categories: Programming Notes

---

很高兴，阿里开源了其内部的**Java**开发手册，简单点说这是一本**Java**开发规范，比方说以前我一直在纠结工具类的命名到底是以**utils**结尾还是以**util**结尾，那同样地，工具类的包名是以**utils**结尾还是以**util**结尾呢？在这本电子书里就给出了很好的说明。再比如定义数组的时候，我们可以这样`String strs[] = new String[5];`也可以这样`String[] strs = new String[5];`，到底哪种方式更好呢？显然是后一种，后一种明确的指定了我们所定义的变量是`String[]`类型。也许你会说，这些都是小问题并不影响我开发，是的，问题不大，但是规范漂亮的代码看起来难道不是更加的赏心悦目吗？把每一次阅读代码的过程看做是品味一杯醇厚的咖啡不是让人觉得更加惬意吗？当然规范的代码带来的好处远不止如此，比如两个竞争性的开源项目，性能特性等差别不大，其中一个是你主导的，那么这时候，如何能让自己的开源项目获得更多的**star**呢？显然代码规范漂亮简洁的项目肯定能获得更加的流行，再比如当你接手离职人员的代码的时候，看着那一坨坨写的像翔一样的代码，你是不是很想骂娘呢？甚至问候一下他们家的女性成员呢？

在看《阿里巴巴Java开发手册》过程中，其在日志规约部分提到
> 应用中不可直接使用日志系统（Log4j、Logback）中的API，而应依赖使用日志框架SLF4J中的API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一

很遗憾，这里面并没有指明对于**Log4j2**日志系统，是否需要使用门面日志框架呢？而我正在使用的就是**Log4j2**。有关**Log4j2**的介绍请参考[这里](http://www.importnew.com/3046.html)和[这里](https://logging.apache.org/log4j/2.x/)，现在的**Log4j2**已经是Java中最优秀的日志框架了，那么我们是否还需要留有余地，以便日后更换日志框架呢？因为使用**slf4j**的主要一个目的就是可以方便的更换底层的具体日志框架，而如果没有更换日志框架的必要的话，那么自然也就没有使用**slf4j**的必要了。

当然了，更多的情况是为了兼容性考虑，比如旧有的项目一直用的都是**slf4j**，那么这时如果想要结合**log4j2**使用的话，上层需要面向**slf4j** API编程，而底层日志框架指定**log4j2**，需要添加如下依赖：

```xml
<!-- log配置：Log4j2的核心依赖 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.8</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.8</version>
</dependency>

<!-- 桥接：告诉Slf4j使用Log4j2 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.8</version>
</dependency>

<!-- 面向slf4j API编程 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.24</version>
</dependency>
```

可以看出，这样需要依赖4个Jar包，而实际上**log4j2**核心的Jar包只有2个。如果**log4j2**已经足够完美，并且我们也不需要切换底层日志框架的话，是不是直接面向**log4j2**的API编程更好呢？秉着一向追求完美的习惯，于是去[stackoverflow](http://stackoverflow.com)上逛了一下，发现已有类似问题。[**Is it worth to use slf4j with log4j2**](http://stackoverflow.com/questions/41498021/is-it-worth-to-use-slf4j-with-log4j2)中就说了，推荐直接面向**log4j2** API编程，理由如下：
- Message API
- Lambdas for lazy logging
- Log any Object instead of just Strings
- Garbage-free: avoid creating varargs or creating Strings where possible
- CloseableThreadContext automatically removes items from the MDC when you're finished with them

而且就目前来说，对于**log4j2**中的诸多特性，**slf4j**并不支持（See [10 Log4j2 API features not available in SLF4J](http://stackoverflow.com/questions/41633278/can-we-use-all-features-of-log4j2-if-we-use-it-along-with-slf4j-api/41635246#41635246) for more details），最重要的是**log4j2**包含了一个`log4j-to-slf4j`模块，该模块可以在任何时候将任何面向**log4j2** API编程的代码转向任何具体的**slf4j**的实现框架，其调用流程简单描述如下：

<div align=center>
![](http://7xig3q.com1.z0.glb.clouddn.com/log4j2_API_slf4j.png)
</div>


综上所述，现在，你可以直接面向**log4j2** API编程了。