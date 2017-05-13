title: "Java中Web Project如何加载dll/DLL文件"
date: 2015-05-19 20:41:23
tags: [DLL, Tomcat]
categories: Programming Notes
---

基本上常用的项目有两种，一种是Java Project，另一种是Web Project，下面就以这两种项目为例，来阐释如何在项目中加载dll文件。
### Java中调用dll的方式

* System.load()
```java 
/**
*Loads the native library specified by the filename argument. The filename argument must be an absolute path name.
*/
public static void load(String filename)
//等价于
Runtime.getRuntime().load(name)
```
由JDK的说明文档可知，load()方法接收的是绝对路径。

* System.loadLibrary() 
```java
/**
Loads the native library specified by the libname argument. The libname argument must not contain any platform specific prefix, file extension or path.
*/
public static void loadLibrary(String libname)
//等价于
Runtime.getRuntime().loadLibrary(name)
```
由JDK的说明文档可知，loadLibrary()方法不接收任何平台相关的特定前缀、文件扩展名或者路径。该方法会自动搜索一些固定路径，我们只需要把dll文件放入**Path**（系统的环境变量）路径下或者**System32**路径下，也可以直接放在Java项目的根目录下。以上几种方法都可以成功加载。
### Java Project
如果是Java Project，只要把dll文件放在Path或者System32或者项目根目录下,基本上都可以正确的加载到dll文件。

### Web Project
根据[2014年最流行的应用服务器](http://www.importnew.com/12590.html)的统计结果，41%的应用服务器部署的是Tomcat，其中出乎我意料的是Jetty以31%的份额占据了第二把交椅，那就以最常用的Tomcat为例吧。

如果使用load()方法加载dll文件，那么代码应该这么写
```java
String path = DBUtils.class.getResource("/").getPath();
path = path.replaceAll("%20", " ");//排除中文空格
System.load(path + "user.dll");  
```

如果使用loadLibrary()方法加载dll文件，简单的如上处理已经不能解决问题了，这是因为Web项目中，将 java.library.path 这个系统属性输出了一下，结果出来两个路径：

* %JAVA_HOME%/bin
* %TOMCAT_HOME%/bin

当然了其实还应该包括一个路径就是

* %JRE_HOME%/bin

所以这时候，在Windows上的Web Project系统至少有三个地方可以放置我们的dll文件，你可以任取其一即可。对于**JAVA_HOME**和**JRE_HOME**，只要放入它们对应的**bin**目录下即可，但是**TOMCAT_HOME**就稍微复杂一点，一定要记住把dll文件放在和启动Tomcat的文件同一目录中，一般来说，放入tomcat的bin目录下即可，这是因为启动tomcat的命令**startup.bat**就在bin目录下，但是如果启动tomcat的命令不在tomcat的bin目录下，dll文件就应该放在启动tomcat的命令所在的目录。

也许有人说，难道有的tomcat的启动命令不是在bin目录下，是的，maybe就真的存在，比如在Eclipse中配置tomcat（免安装的tomcat版本），然后在Eclipse中启动tomcat，这时候，仅仅简单的将dll文件放在tomcat/bin/目录下是行不通的，大家可以测试一下。本人测试在**JAVA_HOME**的bin目录中是完全没问题的，但是这样的话当项目越来越多，dll文件越来越多，就很难区分开哪些dll文件是属于哪些项目的了。
### 路径测试
环境是编译完成的本地LTP动态链接库，这些链接库可能还会加载LTP训练好的一些model-模型文件，所以作为测试不太合适，因为受model不定因素影响，但是可以借鉴一下。

* 在Eclipse中配置的Tomcat情况下，当在类中使用System.loadLibrary("ner_jni");方法时

>将dll文件放在WEB-INF/lib/下面找不到dll文件
将dll文件放在Tomcat/bin/目录下面找不到dll文件
将dll文件放在Web Project的src目录下面找不到dll文件

* 当在类中使用
```java
String str = NER.class.getResource("/").getPath();
str = str.replaceAll("%20", "");
System.load(str + "ner_jni.dll");
```

方式时，并将dll文件放在Web Project的src目录下会出现

>[java.lang.UnsatisfiedLinkError: D:\devSoft\apache-tomcat-7.0.52\webapps\DaseLab\WEB-INF\classes\segmentor_jni.dll: Can't find dependent libraries]

这已经不是找不到dll文件的问题了，而是找不到依赖库，说明这种方式或许是可以加载到dll文件的，但是加载dll文件的顺序不正确，先加载的dll文件可能会依赖于后加载的dll文件，所以这种方式，一定要注意加载dll文件的顺序。解决办法可以参见【4】中的资料。

* 使用loadLibrary()方法时，将dll文件放在Java安装目录的bin/下，这种方式肯定可以加载到dll文件，但是可能会出现

>Classloader, XXX.dll already loaded in another classloader 


参考资料：
【1】http://www.cnblogs.com/zfc2201/archive/2011/09/02/2163268.html
【2】http://www.programgo.com/article/4119403163/
【3】http://blog.csdn.net/zhyh1986/article/details/9199063
【4】http://www.111cn.net/jsp/Java/41523.htm