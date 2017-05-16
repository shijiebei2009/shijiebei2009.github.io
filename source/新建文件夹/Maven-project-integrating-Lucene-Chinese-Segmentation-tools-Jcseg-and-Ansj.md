title: "Maven项目中Lucene集成中文分词工具Jcseg和Ansj"
date: 2016-03-23 22:11:04
tags: [Lucene]
categories: Programming Notes

---

####开发环境
**Windows**：Win7 64 bit
**Java**：java version “1.8.0_45”；Java HotSpot(TM) 64-Bit Server VM
**Lucene**：5.5
**Jcseg**：1.9.7
**Ansj_seg**：3.7.1
**Ansj_lucene5_plug**：3.0
**Maven**：3.3.3

####Lucene集成Jcseg
#####Jcseg简介
**Jcseg**是使用**Java**开发的一个开源中文分词器，使用流行的**mmseg**算法实现，有兴趣的可以参考[算法原文](http://technology.chtsai.org/mmseg/)，并且提供了最高版本的**lucene**、**solr**、**elasticsearch**的分词接口。

**Google Code**最新版**V1.9.6**：https://code.google.com/archive/p/jcseg/
**Git OSChina**最新版**V1.9.7**：http://git.oschina.net/lionsoul/jcseg
**GitHub**地址：https://github.com/lionsoul2014/jcseg

#####Maven编译Jcseg项目源码
从**OSChina**上下载**ZIP**压缩包之后，解压即可，文件夹重命名为**jcseg**，进入**jcseg**根目录，使用**mvn clean package**命令打包，前提是已经配置过**maven**环境变量，否则**mvn**无法识别。成功打包之后如下所示
>[INFO] Reactor Summary:
[INFO]
[INFO] jcseg-core ......................................... SUCCESS [ 13.469 s]
[INFO] jcseg-analyzer ..................................... SUCCESS [  4.601 s]
[INFO] jcseg-elasticsearch ................................ SUCCESS [  4.355 s]
[INFO] jcseg-server ....................................... SUCCESS [  7.346 s]
[INFO] jcseg .............................................. SUCCESS [  0.113 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 30.134 s
[INFO] Finished at: 2016-03-22T16:47:59+08:00
[INFO] Final Memory: 31M/282M
[INFO] ------------------------------------------------------------------------

#####Maven将Jar包安装到本地仓库
打开**maven**的**settings.xml**文件，配置本地仓库路径
```xml
<settings>
    <localRepository>D:/apache-maven-3.3.3/repo</localRepository>
</settings>
```
使用**maven**命令**mvn clean install**将之前打包的**jar**文件安装到本地**maven**仓库中，成功安装如下所示
>[INFO] --- maven-install-plugin:2.4:install (default-install) @ jcseg-server ---
[INFO] Installing D:\jcseg\jcseg-server\target\jcseg-server-1.9.7.jar to D:\apache-maven-3.3.3\repo\org\lionsoul\jcseg\jcseg-server\1.9.7\jcseg-server-1.9.7.jar
[INFO] Installing D:\jcseg\jcseg-server\dependency-reduced-pom.xml to D:\apache-maven-3.3.3\repo\org\lionsoul\jcseg\jcseg-server\1.9.7\jcseg-server-1.9.7.pom
[INFO] Installing D:\jcseg\jcseg-server\target\jcseg-server-1.9.7-sources.jar to D:\apache-maven-3.3.3\repo\org\lionsoul\jcseg\jcseg-server\1.9.7\jcseg-server-1.9.7-sources.jar
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building jcseg 1.9.7
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ jcseg ---
[INFO] 
[INFO] --- maven-enforcer-plugin:1.0:enforce (enforce-maven) @ jcseg ---
[INFO] 
[INFO] --- maven-install-plugin:2.4:install (default-install) @ jcseg ---
[INFO] Installing D:\jcseg\pom.xml to D:\apache-maven-3.3.3\repo\org\lionsoul\jcseg\jcseg\1.9.7\jcseg-1.9.7.pom
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] jcseg-core ......................................... SUCCESS [ 10.195 s]
[INFO] jcseg-analyzer ..................................... SUCCESS [  6.793 s]
[INFO] jcseg-elasticsearch ................................ SUCCESS [  3.815 s]
[INFO] jcseg-server ....................................... SUCCESS [  5.745 s]
[INFO] jcseg .............................................. SUCCESS [  0.016 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 26.687 s
[INFO] Finished at: 2016-03-22T16:53:28+08:00
[INFO] Final Memory: 31M/253M
[INFO] ------------------------------------------------------------------------

#####Maven项目配置Jcseg
如果按照默认添加依赖方式如下所示
```xml
<dependency>
      <groupId>org.lionsoul.jcseg</groupId>
      <artifactId>jcseg-core</artifactId>
      <version>1.9.7</version>
</dependency>
```
是无法成功引用到本地仓库中的**jcseg**的，这是因为，如果依赖的版本是**RELEASE**或者**LATEST**，则**Maven**基于更新策略读取所有远程仓库的元数据**groupId/artifactId/maven-metadata.xml**，将其与本地仓库的对应元数据合并，如果不指定依赖版本，也就是说当依赖版本不明晰的时候，如**RELEASE**、**LATEST**、**SNAPSHOT**，**Maven**就需要基于更新远程仓库的更新策略来检查更新。而我们知道**jcseg**是我们自己安装到本地仓库中的，远程仓库必然不存在对应的元数据文件，所以这时**maven**会报错，那么如何解决呢？

这时候就需要指定**scope**属性，**scope**用来指定依赖的范围，当**scope**指定的值是**system**的时候，**Maven**直接从本地文件系统解析构建，而不会去远程仓库查询。

**scope**还有其它几种取值，说明如下
- **compile**：编译依赖范围，如果没有指定，就会默认使用该依赖范围
- **test**：测试依赖范围，使用此依赖范围的**Maven**依赖，只对于测试**classpath**有效，典型的例子是**JUnit**，它只在编译测试代码及运行测试的时候才需要
- **provided**：已提供依赖范围，使用此依赖范围的**Maven**依赖，对于编译和测试**classpath**有效，但在运行时无效，典型的例子是**servlet-api**，编译和测试项目的时候需要该依赖，但在运行项目的时候，由于容器已经提供，就不需要**Maven**重复地引入一遍
- **runtime**：运行时依赖范围，使用此依赖范围的**Maven**依赖，对于测试和运行**classpath**有效，但在编译主代码时无效。典型的例子是**JDBC**驱动实现，项目主代码的编译只需要**JDK**提供的**JDBC**接口，只有在执行测试或者运行项目的时候才需要实现上述接口的具体**JDBC**驱动
- **system**：系统依赖范围，该依赖与三种**classpath**的关系，和**provided**依赖范围完全一致。但是，在使用**system**范围的依赖时必须通过**systemPath**元素显式地指定依赖文件路径。通常此依赖与本机系统绑定，造成构建的不可移植，应该谨慎使用。
- **import**：导入依赖范围，该依赖范围不会对三种**classpath**产生实际的影响，三种**classpath**指的是，编译**classpath**、测试**classpath**、运行时**classpath**

所以解决方案已经有了，如下所示，其它依赖本地仓库中的**jar**包配置类似：
```xml
<!--在systemPath中亦可指定其它位置，包括相对路径-->
<dependency>
    <groupId>org.lionsoul.jcseg</groupId>
    <artifactId>jcseg-core</artifactId>
    <version>1.9.7</version>
    <scope>system</scope>
    <systemPath>D:/apache-maven-3.3.3/repo/org/lionsoul/jcseg/jcseg-core/1.9.7/jcseg-core-1.9.7.jar</systemPath>
</dependency>
<dependency>
    <groupId>org.lionsoul.jcseg</groupId>
    <artifactId>jcseg-analyzer</artifactId>
    <version>1.9.7</version>
    <scope>system</scope>
    <systemPath>D:/apache-maven-3.3.3/repo/org/lionsoul/jcseg/jcseg-analyzer/1.9.7/jcseg-analyzer-1.9.7.jar</systemPath>
</dependency>
```

#####Lucene集成Jcseg的测试代码
将**jcseg**源码包中的**lexicon**和**jcseg.properties**两个文件复制到**src/main/resources**下，并修改**jcseg.properties**中的`lexicon.path = src/main/resources/lexicon`
```java
@Test
public void test() throws IOException, ParseException {
    String text = "jcseg是使用Java开发的一款开源的中文分词器, 基于流行的mmseg算法实现，分词准确率高达98.4%, 支持中文人名识别, 同义词匹配, 停止词过滤等。并且提供了最新版本的lucene,solr,elasticsearch分词接口。";
    //如果不知道选择哪个Directory的子类，那么推荐使用FSDirectory.open()方法来打开目录
    Analyzer analyzer = new JcsegAnalyzer5X(JcsegTaskConfig.COMPLEX_MODE);
    //非必须(用于修改默认配置): 获取分词任务配置实例
    JcsegAnalyzer5X jcseg = (JcsegAnalyzer5X) analyzer;
    JcsegTaskConfig config = jcseg.getTaskConfig();
    //追加同义词, 需要在 jcseg.properties中配置jcseg.loadsyn=1
    config.setAppendCJKSyn(true);
    //追加拼音, 需要在jcseg.properties中配置jcseg.loadpinyin=1
    config.setAppendCJKPinyin(true);
    //更多配置, 请查看 org.lionsoul.jcseg.tokenizer.core.JcsegTaskConfig
    Directory directory = new RAMDirectory();
    indexWriterConfig = new IndexWriterConfig(analyzer);
    indexWriterConfig.setOpenMode(IndexWriterConfig.OpenMode.CREATE_OR_APPEND);
    indexWriter = new IndexWriter(directory, indexWriterConfig);
    Document document = new Document();
    document.add(new StringField("id", "1000", Field.Store.YES));
    document.add(new TextField("text", text, Field.Store.YES));
    indexWriter.addDocument(document);
    indexWriter.commit();
    IndexReader indexReader = DirectoryReader.open(directory);
    IndexSearcher indexSearcher = new IndexSearcher(indexReader);
    String key = "中文分词器";
    QueryParser queryParser = new QueryParser("text", analyzer);
    queryParser.setDefaultOperator(QueryParser.Operator.AND);
    Query parse = queryParser.parse(key);
    System.out.println(parse);
    TopDocs search = indexSearcher.search(parse, 10);
    System.out.println("命中数：" + search.totalHits);
    ScoreDoc[] scoreDocs = search.scoreDocs;
    for (ScoreDoc sd : scoreDocs) {
        Document doc = indexSearcher.doc(sd.doc);
        System.out.println("得分：" + sd.score);
        System.out.println(doc.get("text"));
    }
}
```
结果输出
>+text:中文 +text:zhong wen +text:国语 +text:分词器 +text:fen ci qi
命中数：1
得分：0.080312796
jcseg是使用Java开发的一款开源的中文分词器, 基于流行的mmseg算法实现，分词准确率高达98.4%, 支持中文人名识别, 同义词匹配, 停止词过滤等。并且提供了最新版本的lucene,solr,elasticsearch分词接口。

####Lucene集成Ansj
#####Ansj简介
**Ansj**是一个**ICTCLAS**的**Java**实现。基本上重写了所有的数据结构和算法。词典是用的开源版的**ICTCLAS**所提供的。并且进行了部分的人工优化。

还是一个基于**n-Gram+**条件随机场模型的中文分词的**Java**实现。分词速度达到每秒钟大约200万字左右（Mac Air下测试），准确率能达到96%以上。目前实现了中文分词、中文姓名识别、用户自定义词典。可以应用到自然语言处理等方面，适用于对分词效果要求高的各种项目。

**GitHub**项目地址：https://github.com/NLPchina/ansj_seg
**Ansj**的仓库地址，包括针对**Lucene**的插件：http://maven.nlpcn.org/org/ansj/

#####Maven项目配置Ansj
根据官方手册，在**pom.xml**文件中加入依赖，如下所示
```xml
<dependency>
    <groupId>org.ansj</groupId>
    <artifactId>ansj_seg</artifactId>
    <version>3.7.1</version>
</dependency>
<dependency>
    <groupId>org.ansj</groupId>
    <artifactId>ansj_lucene5_plug</artifactId>
    <version>3.0</version>
</dependency>
```
但是你应该能想到，中央仓库或者镜像仓库中并没有**Ansj**啊，那上述依赖必然报错。对的，所以还需要在**settings.xml**加入**Ansj**的仓库地址
```xml
<repositories>
  <repository>
    <id>mvn-repo</id>
    <name>ansj maven repo</name>
    <url>http://maven.nlpcn.org/</url>
  </repository>
</repositories>
```
如果你使用了中央仓库的镜像请注意如下内容，如果你没使用镜像请忽略之。

一般在天朝，访问**maven**中央仓库速度是很慢的，所以国内一般会用某个中央仓库的镜像，而我用的是**OSChina**的镜像，在镜像的配置中，如果你使用了通配符，那么需要注意该通配符同样会屏蔽掉**Ansj**仓库的地址，所以需要在通配符之后排除掉**Ansj**的仓库地址
```xml
<mirrors>
      <mirror>
          <id>nexus-osc</id>
          <mirrorOf>*,!mvn-repo</mirrorOf>
          <!--<mirrorOf>*</mirrorOf>如果使用这种方式会屏蔽Ansj仓库-->
          <name>Nexus osc</name>
          <url>http://maven.oschina.net/content/groups/public/</url>
      </mirror>
      <mirror>
          <id>nexus-osc-thirdparty</id>
          <mirrorOf>thirdparty</mirrorOf>
          <name>Nexus osc thirdparty</name>
          <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
      </mirror>
</mirrors>
```
当然如果你没使用通配符，而是指定对**Maven**中央仓库做镜像的话，就无需使用`!mvn-repo`进行排除了，其中**mvn-repo**是**Ansj**仓库的**ID**。指定对**Maven**中央仓库做镜像配置如下：
```xml
<mirrors>
        <mirror>
            <id>nexus-osc</id>
            <mirrorOf>central</mirrorOf>
            <name>Nexus osc</name>
            <url>http://maven.oschina.net/content/groups/public/</url>
        </mirror>
        <mirror>
            <id>nexus-osc-thirdparty</id>
            <mirrorOf>thirdparty</mirrorOf>
            <name>Nexus osc thirdparty</name>
            <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
        </mirror>
</mirrors>
```
#####Maven之镜像
如果仓库**X**可以提供仓库**Y**存储的所有内容，那么就可以认为**X**是**Y**的一个镜像。换句话说，任何一个可以从仓库**Y**获得的构建，都能够从它的镜像中获取。

关于镜像的一个更为常见的用法是结合私服。由于私服可以代理任何外部的公共仓库，因此，对于组织内部的**Maven**用户来说，使用一个私服地址就等于使用了所有需要的外部仓库，这可以将配置集中到私服，从而简化**Maven**本身的配置。

有关镜像的一些配置说明如下：
```xml
<mirrorOf>*</mirrorOf>：匹配所有远程仓库
<mirrorOf>external:*</mirrorOf>：匹配所有远程仓库，使用localhost的除外，使用file://协议的除外。也就是说，匹配所有不在本机上的远程仓库
<mirrorOf>repo1,repo2</mirrorOf>：匹配仓库repo1和repo2，使用逗号分隔多个远程仓库
<mirrorOf>*,!repo1</mirrorOf>：匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除
```
需要注意的是，由于镜像仓库完全屏蔽了被镜像仓库，当镜像仓库不稳定或者停止服务的时候，**Maven**仍将无法访问被镜像仓库，因而将无法下载构建。

#####Maven中自定义变量
通常在依赖一个项目多个组件的时候，为每一个组件单独指定版本号是可以的，但是当升级版本号的时候，就需要对每个组件都做升级，很麻烦，这时就需要自定义变量了。在**pom.xml**中定义如下
```xml
<properties>
    <spring.version>5.2.0</spring.version>
</properties>
```
使用时，直接在**version**中引用标签名即可，例如
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>${spring.version}</version>
</dependency>
```

#####Maven内置变量
**Maven**本身就内置了很多预定义变量，可以直接引用，选取一些举例如下
- 内置属性
    * **${basedir}** represents the directory containing pom.xml
    * **${version}** equivalent to **${project.version}** or **${pom.version}**
    * **${project.basedir}** 同 **${basedir}**
    * **${project.baseUri}** 表示项目文件地址
    * **${maven.build.timestamp}** 表示项目构件开始时间;
    * **${maven.build.timestamp.format}** 表示属性 **${maven.build.timestamp}** 的展示格式，默认值为yyyyMMdd-HHmm
- POM属性
    * **${project.build.directory}** results in the path to your "target" dir, this is the same as **${pom.project.build.directory}**
    * **${project.build.outputDirectory}** results in the path to your "target/classes" dir
    * **${project.name}** refers to the name of the project
    * **${project.version}** refers to the version of the project
    * **${project.build.finalName}** refers to the final name of the file created when the built project is packaged
- Settings文件属性
    * **${settings.localRepository}** refers to the path of the user's local repository
- Java系统属性
    * 使用`mvn help:system`命令可以查看所有的**Java**系统属性
    * `System.getProperties()`可以得到所有的**Java**属性
    * **${user.home}** 表示用户目录
    * **${java.home}** specifies the path to the current JRE_HOME environment use with relative paths to get for example:`<jvm>${java.home}../bin/java.exe</jvm>`
- 环境变量属性
    * **${env.M2_HOME}** returns the Maven2 installation path.
    * **${env.JAVA_HOME}** 表示JAVA_HOME环境变量的值;
- 自定义属性
    * `<properties><my.version>hello</my.version></properties>`则引用 **${my.version}**就会得到值hello

#####Lucene集成Ansj的测试代码
**Ansj In Lucene**的官方参考文档：http://nlpchina.github.io/ansj_seg/

到https://github.com/NLPchina/ansj_seg 下载**ZIP**压缩文件，解压，将其中的**library**文件夹和`library.properties`文件拷贝到**maven**项目下的`src/main/resources`中，修改`library.properties`内容如下
```bash
#redress dic file path
ambiguityLibrary=src/main/resources/library/ambiguity.dic
#path of userLibrary this is default library
userLibrary=src/main/resources/library/default.dic
#path of crfModel
crfModel=src/main/resources/library/crf.model
#set real name
isRealName=true
```

**Lucene**集成**Ansj**测试代码如下
```java
import org.ansj.library.UserDefineLibrary;
import org.ansj.lucene5.AnsjAnalyzer;
import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.tokenattributes.CharTermAttribute;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.index.CorruptIndexException;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.queryparser.classic.ParseException;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.search.highlight.Highlighter;
import org.apache.lucene.search.highlight.InvalidTokenOffsetsException;
import org.apache.lucene.search.highlight.QueryScorer;
import org.apache.lucene.search.highlight.SimpleHTMLFormatter;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.RAMDirectory;
import org.junit.Test;
import org.tartarus.snowball.ext.PorterStemmer;

import java.io.IOException;
import java.io.Reader;
import java.io.StringReader;
import java.util.Date;

public class IndexTest {
    @Test
    public void test() throws IOException {
        Analyzer ca = new AnsjAnalyzer();
        Reader sentence = new StringReader(
                "\n\n\n\n\n\n\n我从小就不由自主地认为自己长大以后一定得成为一个象我父亲一样的画家, 可能是父母潜移默化的影响。其实我根本不知道作为画家意味着什么，我是否喜欢，最重要的是否适合我，我是否有这个才华。其实人到中年的我还是不确定我最喜欢什么，最想做的是什么？我相信很多人和我一样有同样的烦恼。毕竟不是每个人都能成为作文里的宇航员，科学家和大教授。知道自己适合做什么，喜欢做什么，能做好什么其实是个非常困难的问题。");
        TokenStream ts = ca.tokenStream("sentence", sentence);
        System.out.println("start: " + (new Date()));
        long before = System.currentTimeMillis();
        while (ts.incrementToken()) {
            System.out.println(ts.getAttribute(CharTermAttribute.class));
        }
        ts.close();
        long now = System.currentTimeMillis();
        System.out.println("time: " + (now - before) / 1000.0 + " s");
    }

    @Test
    public void indexTest() throws IOException, ParseException {
        Analyzer analyzer = new AnsjAnalyzer();
        Directory directory;
        IndexWriter iwriter;
        String text = "季德胜蛇药片 10片*6板 ";
        UserDefineLibrary.insertWord("蛇药片", "n", 1000);
        IndexWriterConfig ic = new IndexWriterConfig(analyzer);
        // 建立内存索引对象
        directory = new RAMDirectory();
        iwriter = new IndexWriter(directory, ic);
        addContent(iwriter, text);
        iwriter.commit();
        iwriter.close();
        System.out.println("索引建立完毕");
        System.out.println("index ok to search!");
        search(analyzer, directory, "\"季德胜蛇药片\"");
    }

    private void search(Analyzer queryAnalyzer, Directory directory, String queryStr) throws IOException, ParseException {
        IndexSearcher isearcher;
        DirectoryReader directoryReader = DirectoryReader.open(directory);
        // 查询索引
        isearcher = new IndexSearcher(directoryReader);
        QueryParser tq = new QueryParser("text", queryAnalyzer);
        Query query = tq.parse(queryStr);
        System.out.println(query);
        TopDocs hits = isearcher.search(query, 5);
        System.out.println(queryStr + ":共找到" + hits.totalHits + "条记录!");
        for (int i = 0; i < hits.scoreDocs.length; i++) {
            int docId = hits.scoreDocs[i].doc;
            Document document = isearcher.doc(docId);
            System.out.println(toHighlighter(queryAnalyzer, query, document));
        }
    }

    private String toHighlighter(Analyzer analyzer, Query query, Document doc) {
        String field = "text";
        try {
            SimpleHTMLFormatter simpleHtmlFormatter = new SimpleHTMLFormatter("<font color=\"red\">", "</font>");
            Highlighter highlighter = new Highlighter(simpleHtmlFormatter, new QueryScorer(query));
            TokenStream tokenStream1 = analyzer.tokenStream("text", new StringReader(doc.get(field)));
            String highlighterStr = highlighter.getBestFragment(tokenStream1, doc.get(field));
            return highlighterStr == null ? doc.get(field) : highlighterStr;
        } catch (IOException | InvalidTokenOffsetsException e) {
        }
        return null;
    }

    private void addContent(IndexWriter iwriter, String text) throws CorruptIndexException, IOException {
        Document doc = new Document();
        doc.add(new Field("text", text, Field.Store.YES, Field.Index.ANALYZED));
        iwriter.addDocument(doc);
    }

    @Test
    public void poreterTest() {
        PorterStemmer ps = new PorterStemmer();
        System.out.println(ps.stem());
    }

}
```
本以为大功告成，怀着激动的心情运行**Junit**单元测试，但是天雷滚滚啊，报错啊，想想这可是官方给的测试**Demo**啊，居然报错！！！

废话不说，上结果：
>**java.lang.AssertionError: TokenStream implementation classes or at least their incrementToken() implementation must be final**

**Notes 2016/3/27**：经反馈，作者已**fix**此**Bug**，但本文不再做更新。**Issue**编号：[#249](https://github.com/NLPchina/ansj_seg/issues/249)，反馈地址：https://github.com/NLPchina/ansj_seg/issues/249#event-604309598

说的很清楚啊，**Ansj**作者自己测试通过否？要么将类修饰为**final**，要么将**incrementToken()**方法修饰为**final**，这是为啥哩？

查源码，从**AnsjAnalyzer**追踪到**AnsjTokenizer**，再追踪到**Tokenizer**，别停，继续追踪**TokenStream**，该类位于`org.apache.lucene.analysis`包下，在该类的**Doc**注释中，终于发现如下说明
> The {@code TokenStream}-API in Lucene is based on the decorator pattern. Therefore all non-abstract subclasses must be final or have at least a final implementation of {@link #incrementToken}! This is checked when Java assertions are enabled.

意思是说所有的非抽象子类必须是**final**的或者至少有一个**final**修饰的**incrementToken()**覆写方法。但是**Ansj**针对**Lucene**的插件中，这两者都没有做！！！

#####Lucene集成Ansj报错解决方案

1. 既然**Ansj**两者都没做，那么一种方法就是修改**Ansj**的源码，但是我们使用的是**Ansj**仓库中提供的**Jar**包，修改源码之后，只能本地引用修改后的**Jar**包，不方便项目的迁移，所以不采用
2. 提供一个**final**修饰的覆写方法**incrementToken()**，通过实现两个内部类，分别继承自**AnsjAnalyzer**和**AnsjTokenizer**，在使用的时候调用自己实现的内部类

修复**BUG**后的代码如下
```java
import org.ansj.library.UserDefineLibrary;
import org.ansj.lucene.util.AnsjTokenizer;
import org.ansj.lucene5.AnsjAnalyzer;
import org.ansj.splitWord.Analysis;
import org.ansj.splitWord.analysis.IndexAnalysis;
import org.ansj.splitWord.analysis.ToAnalysis;
import org.ansj.splitWord.analysis.UserDefineAnalysis;
import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.Tokenizer;
import org.apache.lucene.analysis.tokenattributes.CharTermAttribute;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.index.CorruptIndexException;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.queryparser.classic.ParseException;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.search.highlight.Highlighter;
import org.apache.lucene.search.highlight.InvalidTokenOffsetsException;
import org.apache.lucene.search.highlight.QueryScorer;
import org.apache.lucene.search.highlight.SimpleHTMLFormatter;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.RAMDirectory;
import org.junit.Test;
import org.nlpcn.commons.lang.tire.domain.Forest;
import org.tartarus.snowball.ext.PorterStemmer;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.Reader;
import java.io.StringReader;
import java.util.Date;
import java.util.Set;

public class IndexTest {
    @Test
    public void test() throws IOException {
        Analyzer ca = new MyAnsjAnalyzer();
        Reader sentence = new StringReader(
                "\n\n\n\n\n\n\n我从小就不由自主地认为自己长大以后一定得成为一个象我父亲一样的画家, 可能是父母潜移默化的影响。其实我根本不知道作为画家意味着什么，我是否喜欢，最重要的是否适合我，我是否有这个才华。其实人到中年的我还是不确定我最喜欢什么，最想做的是什么？我相信很多人和我一样有同样的烦恼。毕竟不是每个人都能成为作文里的宇航员，科学家和大教授。知道自己适合做什么，喜欢做什么，能做好什么其实是个非常困难的问题。");
        TokenStream ts = ca.tokenStream("sentence", sentence);
        System.out.println("start: " + (new Date()));
        long before = System.currentTimeMillis();
        while (ts.incrementToken()) {
            System.out.println(ts.getAttribute(CharTermAttribute.class));
        }
        ts.close();
        long now = System.currentTimeMillis();
        System.out.println("time: " + (now - before) / 1000.0 + " s");
    }

    static class MyAnsjTokenizer extends AnsjTokenizer {
        public MyAnsjTokenizer(Analysis ta, Set<String> filter) {
            super(ta, filter);
        }

        @Override
        public final boolean incrementToken() throws IOException {
            return super.incrementToken();
        }
    }

    static class MyAnsjAnalyzer extends AnsjAnalyzer {
        private Set<String> filter;
        private String type;

        public MyAnsjAnalyzer() {
        }

        public MyAnsjAnalyzer(String type, Set<String> filter) {
            super(type, filter);
        }

        @Override
        protected TokenStreamComponents createComponents(String text) {
            BufferedReader reader = new BufferedReader(new StringReader(text));
            Tokenizer tokenizer = getTokenizer(reader, this.type, this.filter);
            return new TokenStreamComponents(tokenizer);
        }

        public static Tokenizer getTokenizer(BufferedReader reader, String type, Set<String> filter) {
            AnsjTokenizer tokenizer;
            if ("user".equalsIgnoreCase(type)) {
                tokenizer = new MyAnsjTokenizer(new UserDefineAnalysis(reader, new Forest[0]), filter);
            } else if ("index".equalsIgnoreCase(type)) {
                tokenizer = new MyAnsjTokenizer(new IndexAnalysis(reader, new Forest[0]), filter);
            } else {
                tokenizer = new MyAnsjTokenizer(new ToAnalysis(reader, new Forest[0]), filter);
            }

            return tokenizer;
        }
    }

    @Test
    public void indexTest() throws IOException, ParseException {
        Analyzer analyzer = new MyAnsjAnalyzer();
        Directory directory;
        IndexWriter iwriter;
        String text = "季德胜蛇药片 10片*6板 ";
        UserDefineLibrary.insertWord("蛇药片", "n", 1000);
        IndexWriterConfig ic = new IndexWriterConfig(analyzer);
        // 建立内存索引对象
        directory = new RAMDirectory();
        iwriter = new IndexWriter(directory, ic);
        addContent(iwriter, text);
        iwriter.commit();
        iwriter.close();
        System.out.println("索引建立完毕");
        System.out.println("index ok to search!");
        search(analyzer, directory, "\"季德胜蛇药片\"");
    }

    private void search(Analyzer queryAnalyzer, Directory directory, String queryStr) throws IOException, ParseException {
        IndexSearcher isearcher;
        DirectoryReader directoryReader = DirectoryReader.open(directory);
        // 查询索引
        isearcher = new IndexSearcher(directoryReader);
        QueryParser tq = new QueryParser("text", queryAnalyzer);
        Query query = tq.parse(queryStr);
        System.out.println(query);
        TopDocs hits = isearcher.search(query, 5);
        System.out.println(queryStr + ":共找到" + hits.totalHits + "条记录!");
        for (int i = 0; i < hits.scoreDocs.length; i++) {
            int docId = hits.scoreDocs[i].doc;
            Document document = isearcher.doc(docId);
            System.out.println(toHighlighter(queryAnalyzer, query, document));
        }
    }

    private String toHighlighter(Analyzer analyzer, Query query, Document doc) {
        String field = "text";
        try {
            SimpleHTMLFormatter simpleHtmlFormatter = new SimpleHTMLFormatter("<font color=\"red\">", "</font>");
            Highlighter highlighter = new Highlighter(simpleHtmlFormatter, new QueryScorer(query));
            TokenStream tokenStream1 = analyzer.tokenStream("text", new StringReader(doc.get(field)));
            String highlighterStr = highlighter.getBestFragment(tokenStream1, doc.get(field));
            return highlighterStr == null ? doc.get(field) : highlighterStr;
        } catch (IOException | InvalidTokenOffsetsException e) {
        }
        return null;
    }

    private void addContent(IndexWriter iwriter, String text) throws CorruptIndexException, IOException {
        Document doc = new Document();
        doc.add(new Field("text", text, Field.Store.YES, Field.Index.ANALYZED));
        iwriter.addDocument(doc);
    }

    @Test
    public void poreterTest() {
        PorterStemmer ps = new PorterStemmer();
        System.out.println(ps.stem());
    }

}
```
运行部分结果如下：
>信息: init user userLibrary ok path is : D:\Multi-module-project\Lucene\src\main\resources\library\default.dic
信息: init ambiguityLibrary ok!
信息: init core library ok use time :281
信息: init ngram ok use time :668
索引建立完毕
index ok to search!
text:"季德胜 蛇药片"
"季德胜蛇药片":共找到1条记录!
<font color="red">季德胜</font><font color="red">蛇药片</font> 10片*6板

#####Ansj设置词典路径
1. 正规方式
创建**library.properties**中增加
```bash
#path of userLibrary 自定义词典路径
userLibrary=library/userLibrary/userLibrary.dic
ambiguityLibrary=library/ambiguity.dic
```

2. 在用词典未加载前可以通过，代码方式方式来加载
```java
MyStaticValue.userLibrary=[你的路径]
```

3. 调用api加载。在程序运行的任何时间都可以。动态加载。
```java
loadLibrary.loadLibrary(String path)方式加载
```
路径可以是具体文件也可以是一个目录，如果是一个目录，那么会扫描目录下的**dic**文件自动加入词典。

#####Lucene集成Ansj添加自定义词典
如果需要添加自己的自定义词典，参考**default.dic**格式即可。用户自定义词典的格式是`word[tab]nature[tab]freq`，**example**: 小李子 nr 100

####分布式分词组件Word
***Notes***：另外发现了一个分布式中文分词组件，希望以后有机会可以深入研究，地址：https://github.com/ysc/word

**Word**分词是一个**Java**实现的分布式的中文分词组件，提供了多种基于词典的分词算法，并利用**ngram**模型来消除歧义。能准确识别英文、数字，以及日期、时间等数量词，能识别人名、地名、组织机构名等未登录词。能通过自定义配置文件来改变组件行为，能自定义用户词库、自动检测词库变化、支持大规模分布式环境，能灵活指定多种分词算法，能使用**refine**功能灵活控制分词结果，还能使用词频统计、词性标注、同义标注、反义标注、拼音标注等功能。提供了10种分词算法，还提供了10种文本相似度算法，同时还无缝和**Lucene**、**Solr**、**ElasticSearch**、**Luke**集成。注意：**word1.3**需要**JDK1.8**。