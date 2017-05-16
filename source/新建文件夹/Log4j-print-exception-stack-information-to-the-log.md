title: "Log4j打印异常堆栈信息到日志"
date: 2016-04-15 23:38:41
tags: [Log4j]
categories: Programming Notes
toc: false

---
<blockquote  class="blockquote-center">
**把辛勤的耕作当做生命的必要，即使没有收获的指望依然心平气和的继续耕种。**

路遥
</blockquote>

在**Java**中，通常情况下，需要将异常堆栈信息输出到日志中，这样便于纠错及修正**Bug**，而多数情况下，大家最常用的是使用`e.printStackTrace()`直接打印堆栈信息完事，这并不是值的推荐的做法。

1. 当出现异常时，调用`e.printStackTrace();`其实相当于什么都没做，同时也不会把异常信息输出到日志文件中
2. 使用`log.error(e.getMessage());`只能够输出异常信息，但是并不包括异常堆栈，所以无法追踪出错的源点
3. 使用`log.error(e);`除了输出异常信息外，还能输出异常类型，但是同样不包括异常堆栈，该方法**doc**说明为：**Logs a message object with the ERROR level.**显然并不会记录异常堆栈信息
4. 当然也可以自己手动写个工具类，来挨个输出`e.getStackTrace();`获得的堆栈信息，显然繁琐麻烦
5. 其实在**log4j**中只需要这样调用，就可以获得异常及堆栈信息`log.error(Object var1, Throwable var2);`，该方法**doc**说明为：**Logs a message at the ERROR level including the stack trace of the Throwable t passed as parameter.**

**例如：**
```java
    @Test
    public void testError() {
        String s = null;
        try {
            s.length();
        } catch (Exception e) {
            log.error(s, e);
            //当然如果你懒得想提示信息的话，直接这样log.error("", e);
        }
    }
```
**输出：**
>2016.04.14 12:41:56 GMT+08:00 ERROR XX XX testError - null java.lang.NullPointerException at XX(XX.java:YY) [classes/:?]
......
......
