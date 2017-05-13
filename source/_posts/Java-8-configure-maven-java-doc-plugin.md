title: 'Java 8 配置Maven-javadoc-plugin'
date: 2016-10-18 22:41:30
tags: [Maven]
categories: Programming Notes
toc: false

---

在升级JDK至1.8之后，使用`Maven-javadoc-plugin`插件打包报错，**[ERROR] Failed to execute goal org.apache.maven.plugins:maven-javadoc-plugin:2.10.4:jar (attach-javadocs) on project
**详细信息如下

>[ERROR] Failed to execute goal org.apache.maven.plugins:maven-javadoc-plugin:2.10.4:jar (attach-javadocs) on project StatisticsReport: MavenReportException: Error while generating Javadoc:
[ERROR] Exit code: 1 - D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:29: 警告: @param 没有说明
[ERROR] * @param preparedStatement
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:30: 警告: @param 没有说明
[ERROR] * @param params
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:31: 警告: @return 没有说明
[ERROR] * @return
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:32: 警告: @throws 没有说明
[ERROR] * @throws SQLException
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:34: 警告: logFlag没有 @param
[ERROR] public static ResultSet pullData(PreparedStatement preparedStatement, boolean logFlag, String... params) throws SQLException {
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:51: 警告: @param 没有说明
[ERROR] * @param preparedStatement
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:52: 警告: @param 没有说明
[ERROR] * @param params
[ERROR] ^
[ERROR] D:\Multi-module-project\StatisticsReport\src\main\java\com\yuewen\statistics\report\service\db\PullData.java:53: 警告: @return 没有说明
[ERROR] * @return
[ERROR] ^

经查得知，在JDK 8中，Javadoc中添加了*doclint*，而这个工具的主要目的是旨在获得符合W3C HTML 4.01标准规范的HTML文档，在JDK 8中，已经无法获取如下的Javadoc，除非它满足*doclint*：
* 不能有自关闭的HTML tags，例如`<br/>`或者`<a id="x"/>`
* 不能有未关闭的HTML tags，例如有`<ul>`而没有`</ul>`
* 不能有非法的HTML end tags，例如`</br>`
* 不能有非法的HTML attributes，需要符合doclint基于W3C HTML 4.01的实现
* 不能有重复的HTML id attribute
* 不能有空的HTML href attribute
* 不能有不正确的嵌套标题，例如类的文档说明中必须有`<h3>`而不是`<h4>`
* 不能有非法的HTML tags，例如`List<String>`需要用`<>`对应的实体符号
* 不能有损坏的`@link references`
* 不能有损坏的`@param references`，它们必须匹配实际的参数名称
* 不能有损坏的`@throws references`，第一个词必须是一个类名称

注意违反这些规则的话，将不会得到Javadoc的输出。

一种解决办法就是关闭*doclint*，如果你在Maven中运行，你需要使用`additionalparam`设置：
```xml
<profiles>
    <profile>
        <id>disable-javadoc-doclint</id>
        <activation>
            <jdk>[1.8,)</jdk>
        </activation>
        <properties>
            <additionalparam>-Xdoclint:none</additionalparam>
        </properties>
    </profile>
</profiles>
```
或者是添加到`maven-javadoc-plugin`中：
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>2.10.4</version>
    <configuration>
        <encoding>${chartset.UTF8}</encoding>
        <aggregate>true</aggregate>
        <charset>${chartset.UTF8}</charset>
        <docencoding>${chartset.UTF8}</docencoding>
    </configuration>
    <executions>
        <execution>
            <id>attach-javadocs</id>
            <phase>package</phase>
            <goals>
                <goal>jar</goal>
            </goals>
            <configuration>
                <additionalparam>-Xdoclint:none</additionalparam>
            </configuration>
        </execution>
    </executions>
</plugin>
```