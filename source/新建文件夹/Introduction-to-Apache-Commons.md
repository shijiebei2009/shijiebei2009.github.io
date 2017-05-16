title: "Apache Commons入门"
date: 2015-04-19 20:29:21
tags: [Java]
categories: Programming Notes
toc: false

---
Apache Commons是Apache软件基金会的项目，曾隶属于Jakarta项目。Commons的目的是提供可重用的、开源的Java代码。Commons由三部分组成：Proper（是一些已发布的项目）、Sandbox（是一些正在开发的项目）和Dormant（是一些刚启动或者已经停止维护的项目）。

目前Commons-IO包稳定版本是Version 2.4，可惜的是，对于我目前很需要的copyInputStreamToFile（final InputStream source, final File destination, boolean closeSource）方法，只能等到Version 2.5了，关于详情参见：https://issues.apache.org/jira/browse/IO-381

Apache Commons提供了全方位可重用的Java组件，在我们日常开发中，诸多问题的解决方法均可在Commons包中找到实现，使用Commons包提供的组件可以极大的提高开发效率，减少重复劳动，下表是Commons提供的组件的详细信息，鉴于英文水平一般，不予翻译，采用官网提供的标准信息。

![][1]
![Apache commons][2]

[1]: http://7xig3q.com1.z0.glb.clouddn.com/commons-a.png
[2]: http://7xig3q.com1.z0.glb.clouddn.com/commons-b.png
  
  
以最常用的Commons包封装的方法为例，介绍一些简单的使用示例
+ 将输入流转换成文本

```java
package edu.shu.commons.io.test;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;

import org.apache.commons.io.IOUtils;

public class TestCommonsIO {
public static void main(String[] args) throws IOException {
URL url = new URL("http://www.baidu.com");
InputStream openStream = url.openStream();
String string = IOUtils.toString(openStream, "UTF-8");
System.out.println(string);
}
}
```

+ 读文件与写文件

```java
package edu.shu.commons.io.test;

import java.io.File;
import java.io.IOException;

import org.apache.commons.io.FileUtils;

public class TestCommonsIO {
public static void main(String[] args) throws IOException {
String content = FileUtils.readFileToString(new File("test.txt"));
System.out.println(content);
FileUtils.writeStringToFile(new File("destination.txt"), content);
// 指定编码的读
content = FileUtils.readFileToString(new File("test.txt"), "UTF-8");
// 指定编码的写
FileUtils.writeStringToFile(new File("destination.txt"), content, "UTF-8");
}
}
```

+ 输出结果为HTML

```java
package edu.shu.commons.io.test;

import java.io.IOException;

import org.apache.commons.lang3.StringEscapeUtils;

public class TestCommonsIO {
public static void main(String[] args) throws IOException {
String content = "";
String escapeHtml4 = StringEscapeUtils.escapeHtml4(content);
System.out.println(escapeHtml4);

}
}
```

+ 将文本按行读取并以行为元素存入List列表

```java
package edu.shu.commons.io.test;

import java.io.File;
import java.io.IOException;
import java.util.List;

import org.apache.commons.io.FileUtils;

public class TestCommonsIO {
public static void main(String[] args) throws IOException {
List contents= FileUtils.readLines(new File("test.txt"));
System.out.println(contents);
}
}
```

+ 从ClassLoader加载器加载资源，并且读出文本内容、

```java
package edu.shu.commons.io.test;

import java.io.IOException;
import java.io.InputStream;

import org.apache.commons.io.IOUtils;

public class TestCommonsIO {
public static void main(String[] args) throws IOException {
InputStream is = TestCommonsIO.class.getClass().getResourceAsStream("/resource/d.txt");
String content = IOUtils.toString(is);
System.out.println(content);
}
}
```

+ commons mail包主要是对java mail的封装，可以方便快速的发送邮件

```java
import org.apache.commons.mail.DefaultAuthenticator;
import org.apache.commons.mail.Email;
import org.apache.commons.mail.EmailException;
import org.apache.commons.mail.SimpleEmail;

public class TestEmail {
public static void main(String args[]) throws EmailException {
Email email = new SimpleEmail();
email.setHostName("smtp.qq.com");
email.setSmtpPort(465);
email.setAuthenticator(new DefaultAuthenticator("发送者账号", "发送者账号密码"));
email.setSSLOnConnect(true);
email.setFrom("sender mail@qq.com");
email.setSubject("TestMail");
email.setMsg("This is a test mail ... :-)");
email.addTo("receiver mail@qq.com");
email.send();
}
}
```
参考资料：
【1】 http://zh.wikipedia.org/wiki/Apache_Commons
【2】 http://zhoualine.iteye.com/blog/1770014