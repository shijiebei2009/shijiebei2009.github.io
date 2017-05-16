title: Lucene 6.0 实战-索引热备份及恢复
date: 2016-06-24 23:24:43
tags: [Lucene]
categories: Programming Notes

---

###索引备份的几个关键问题
- 最简单的备份方式是关闭IndexWriter，然后逐一拷贝索引文件，但是如果索引比较大，那么这种备份操作会持续较长时间，而在备份期间，程序无法对索引文件进行修改，很多搜索程序是不能接受索引操作期间如此长时间停顿的
- 那么不关闭IndexWriter又如何呢？这样也不行，因为在拷贝索引期间，如果索引文件发生变化，会导致备份的索引文件损坏
- 另外一个问题就是如果原索引文件损坏的话，再备份它也毫无意义，所以一定要备份的是最后一次成功commit之后的索引文件
- 每次在备份之前，如果程序将要覆盖上一个备份，需要先删除备份中未出现在当前快照中的文件，因为这些文件已经不会被当前索引引用了；如果每次都更改备份路径的话，那么就直接拷贝即可

###索引热备份
从Lucene 2.3版本开始，Lucene提供了一个热备策略，就是SnapshotDeletionPolicy，这样就能在不关闭IndexWriter的情况下，对程序最近一次索引修改提交操作时的文件引用进行备份，从而能建立一个连续的索引备份镜像。那么你也许会有疑问，在备份期间，索引出现变化怎么办呢？这就是SnapshotDeletionPolicy的牛逼之处，在使用SnapshotDeletionPolicy.snapshot()获取快照之后，索引更新提交时刻的所有文件引用都不会被IndexWriter删除，只要IndexWriter并未关闭，即使IndexWriter在进行更新、优化操作等也不会删除这些文件。如果说索引拷贝过程耗时较长也不会出现问题，因为被拷贝的文件时索引快照，在快照的有效期，其引用的文件会一直存在于磁盘上。

所以在备份期间，索引会比通常情况下占用更大的磁盘空间，当索引备份完成后，可以调用SnapshotDeletionPolicy.release (IndexCommit commit) 释放指定的某次提交，以便让IndexWriter删除这些已被关闭或下次将要更新的文件。

需要注意的是，Lucene对索引文件的写入操作是一次性完成的。这意味着你可以简单通过文件名比对来完成对索引的增量备份备份，你不必查看每个文件的内容，也不必查看该文件上次被修改的时间戳，因为一旦程序从快照中完成文件写入和引用操作，这些文件就不会改变了。

segments.gen文件在每次程序提交索引更新时都会被重写，因此备份模块必须要经常备份该文件，但是在Lucene 6.0中注意segments.gen已经从Lucene索引文件格式中移除，所以无需单独考虑segments.gen的备份策略了。在备份期间，write.lock文件不用拷贝。

SnapshotDeletionPolicy类有两个使用限制
- 该类在同一时刻只保留一个可用的索引快照，当然你也可以解除该限制，方法是通过建立对应的删除策略来同时保留多个索引快照
- 当前快照不会保存到硬盘，这意味着你关闭旧的IndexWriter并打开一个新的IndexWriter，快照将会被删除，因此在备份结束前是不能关闭IndexWriter的，否则也会报org.apache.lucene.store.AlreadyClosedException: this IndexWriter is closed异常。不过该限制也是很容易解除的：你可以将当前快照存储到磁盘上，然后在打开新的IndexWriter时将该快照保护起来，这样就能在关闭旧的IndexWriter和打开新IndexWriter时继续进行备份操作

索引热备份解决方案
```java
import org.apache.commons.io.FileUtils;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.search.TermQuery;
import org.apache.lucene.store.FSDirectory;

import java.io.File;
import java.io.IOException;
import java.nio.file.Paths;
import java.util.Collection;

import static org.apache.lucene.document.Field.Store.YES;

/**
 * 测试Lucene索引热备
 */
public class TestIndexBackupRecovery {

    public static void main(String[] args) throws IOException, InterruptedException {
        String f = "D:/index_test";
        String d = "D:/index_back";
        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new StandardAnalyzer());
        indexWriterConfig.setIndexDeletionPolicy(new SnapshotDeletionPolicy(new KeepOnlyLastCommitDeletionPolicy()));
        IndexWriter writer = new IndexWriter(FSDirectory.open(Paths.get(f)), indexWriterConfig);
        Document document = new Document();
        document.add(new StringField("ID", "111", YES));
        document.add(new IntPoint("age", 111));
        document.add(new StoredField("age", 111));
        writer.addDocument(document);
        writer.commit();
        document = new Document();
        document.add(new StringField("ID", "222", Field.Store.YES));
        document.add(new IntPoint("age", 333));
        document.add(new StoredField("age", 333));
        writer.addDocument(document);
        document.add(new StringField("ID", "333", Field.Store.YES));
        document.add(new IntPoint("age", 555));
        document.add(new StoredField("age", 555));
        for (int i = 0; i < 1000; i++) {
            document = new Document();
            document.add(new StringField("ID", "333", YES));
            document.add(new IntPoint("age", 1000000 + i));
            document.add(new StoredField("age", 1000000 + i));
            document.add(new StringField("desc", "ABCDEFG" + i, YES));
            writer.addDocument(document);
        }
        writer.deleteDocuments(new TermQuery(new Term("ID", "333")));
        writer.commit();
        backupIndex(writer, f, d);
        writer.close();
    }

    public static void backupIndex(IndexWriter indexWriter, String indexDir, String
            backupIndexDir) throws IOException {
        IndexWriterConfig config = (IndexWriterConfig) indexWriter.getConfig();
        SnapshotDeletionPolicy snapshotDeletionPolicy = (SnapshotDeletionPolicy) config.getIndexDeletionPolicy();
        IndexCommit snapshot = snapshotDeletionPolicy.snapshot();
        //设置索引提交点，默认是null，会打开最后一次提交的索引点
        config.setIndexCommit(snapshot);
        Collection<String> fileNames = snapshot.getFileNames();
        File[] dest = new File(backupIndexDir).listFiles();
        String sourceFileName;
        String destFileName;
        if (dest != null && dest.length > 0) {
            //先删除备份文件中的在此次快照中已经不存在的文件
            for (File file : dest) {
                boolean flag = true;
                //包括文件扩展名
                destFileName = file.getName();
                for (String fileName : fileNames) {
                    sourceFileName = fileName;
                    if (sourceFileName.equals(destFileName)) {
                        flag = false;
                        break;//跳出内层for循环
                    }
                }
                if (flag) {
                    file.delete();//删除
                }
            }
            //然后开始备份快照中新生成的文件
            for (String fileName : fileNames) {
                boolean flag = true;
                sourceFileName = fileName;
                for (File file : dest) {
                    destFileName = file.getName();
                    //备份中已经存在无需复制，因为Lucene索引是一次写入的，所以只要文件名相同不要要hash检查就可以认为它们的数据是一样的
                    if (destFileName.equals(sourceFileName)) {
                        flag = false;
                        break;
                    }
                }
                if (flag) {
                    File from = new File(indexDir + File.separator + sourceFileName);//源文件
                    File to = new File(backupIndexDir + File.separator + sourceFileName);//目的文件
                    FileUtils.copyFile(from, to);
                }
            }
        } else {
            //备份不存在，直接创建
            for (String fileName : fileNames) {
                File from = new File(indexDir + File.separator + fileName);//源文件
                File to = new File(backupIndexDir + File.separator + fileName);//目的文件
                FileUtils.copyFile(from, to);
            }
        }
        snapshotDeletionPolicy.release(snapshot);
        //删除已经不再被引用的索引提交记录
        indexWriter.deleteUnusedFiles();
    }
}
```

###索引文件格式
首先索引里都存了些什么呢？一个索引包含一个documents的序列，一个document是一个fields的序列，一个field是一个有名的terms序列，一个term是一个比特序列。

根据[Summary of File Extensions](https://lucene.apache.org/core/6_0_0/core/index.html?org/apache/lucene/codecs/lucene60/package-summary.html)的说明，目前Lucene 6.0中存在的索引格式如下

|Name|Extension|Brief Description|
|---|---|---|
|Segments File|segments_N|Stores information about a commit point|
|Lock File|write.lock|The Write lock prevents multiple IndexWriters from writing to the same file|
|Segment Info|.si|Stores metadata about a segment|
|Compound File|.cfs, .cfe|An optional "virtual" file consisting of all the other index files for systems that frequently run out of file handles|
|Fields|.fnm|Stores information about the fields|
|Field Index|.fdx|Contains pointers to field data|
|Field Data|.fdt|The stored fields for documents|
|Term Dictionary|.tim|The term dictionary, stores term info|
|Term Index|.tip|The index into the Term Dictionary|
|Frequencies|.doc|Contains the list of docs which contain each term along with frequency|
|Positions|.pos|Stores position information about where a term occurs in the index|
|Payloads|.pay|Stores additional per-position metadata information such as character offsets and user payloads|
|Norms|.nvd, .nvm|Encodes length and boost factors for docs and fields|
|Per-Document Values|.dvd, .dvm|Encodes additional scoring factors or other per-document information|
|Term Vector Index|.tvx|Stores offset into the document data file|
|Term Vector Documents|.tvd|Contains information about each document that has term vectors|
|Term Vector Fields|.tvf|The field level info about term vectors|
|Live Documents|.liv|Info about what files are live|
|Point values|.dii, .dim|Holds indexed points, if any|

在Lucene索引结构中，既保存了正向信息，也保存了反向信息。
正向信息存储在：段(segments_N)->field(.fnm/.fdx/.fdt)->term(.tvx/.tvd/.tvf)
反向信息存储在：词典(.tim)->倒排表(.doc/.pos)

如果想要查看中文，可以参考[这里](http://my.oschina.net/rickylau/blog/527602)。

###恢复索引
恢复索引步骤如下
- 关闭索引目录下的全部reader和writer，这样才能进行文件恢复。对于Windows系统来说，如果还有其它进程在使用这些文件，那么备份程序仍然不能覆盖这些文件
- 删除当前索引目录下的所有文件，如果删除过程出现“访问被拒绝”（Access is denied）错误，那么再次检查上一步是否已完成
- 从备份目录中拷贝文件至索引目录。程序需要保证该拷贝操作不会碰到任何错误，如磁盘空间已满等，因为这些错误会损坏索引文件
- 对于损坏索引，可以使用CheckIndex（org.apache.lucene.index）进行检查并修复

通常Lucene能够很好地避免大多数常见错误，如果程序遇到磁盘空间已满或者OutOfMemoryException异常，那么它只会丢失内存缓冲中的文档，已经编入索引的文档将会完好保存下来，并且索引也会保持原样。这个结果对于以下情况同样适用：如出现JVM崩溃，或者程序碰到无法控制的异常，或者程序进程被终止，或者操作系统崩溃，或者计算机突然断电等。
```java
/**
 * @param source 索引源
 * @param dest 索引目标
 * @param indexWriterConfig 配置相关
 */public static void recoveryIndex(String source, String dest, IndexWriterConfig indexWriterConfig) {
    IndexWriter indexWriter = null;
 try {
        indexWriter = new IndexWriter(FSDirectory.open(Paths.get(dest)), indexWriterConfig);
  } catch (IOException e) {
        log.error("", e);
  } finally {
        //说明IndexWriter正常打开了，无需恢复
  if (indexWriter != null && indexWriter.isOpen()) {
            try {
                indexWriter.close();
  } catch (IOException e) {
                log.error("", e);
  }
        } else {
            //说明IndexWriter已经无法打开，使用备份恢复索引
 //此处简单操作，先清空损坏的索引文件目录，如果索引特别大，可以比对每个文件，不必全部删除  try {
                FileUtils.deleteDirectory(new File(dest));
  FileUtils.copyDirectory(new File(source), new File(dest));
  } catch (IOException e) {
                log.error("", e);
  //使用备份恢复出错，那么就使用最后一招修复索引
  log.info("Check index {} now!", dest);
 try {
                    IndexUtils.checkIndex(dest);
  } catch (IOException | InterruptedException e1) {
                    log.error("Check index error!", e1);
  }
            }
        }
    }
}
```

###修复索引
当其它所有方法都无法解决索引损坏问题时，你的最后一个选项就是使用CheckIndex工具了。该工具除了能汇报索引细节状况以外，还能完成修复索引的工作。该工具会强制删除索引中出现问题的段，需要注意的是，该操作还会全部删除这些段包含的文档，该工具的使用目标应主要着眼于能够在紧急状况下让搜索程序再次运行起来，一旦我们进行了索引备份，并且备份完好，应优先使用恢复索引，而不是修复索引。
```java
/**
 * CheckIndex会检查索引中的每个字节，所以当索引比较大时，此操作会比较耗时
 *
 * @throws IOException
 * @throws InterruptedException
 */
public void checkIndex(String indexFilePath) throws IOException, InterruptedException {
    CheckIndex checkIndex = new CheckIndex(FSDirectory.open(Paths.get(indexFilePath)));
    checkIndex.setInfoStream(System.out);
    CheckIndex.Status status = checkIndex.checkIndex();
    if (status.clean) {
        System.out.println("Check Index successfully！");
    } else {
        //产生索引中的某个文件之后再次测试
        System.out.println("Starting repair index files...");
        //该方法会向索引中写入一个新的segments文件，但是并不会删除不被引用的文件，除非当你再次打开IndexWriter才会移除不被引用的文件
        //该方法会移除所有存在错误段中的Document索引文件
        checkIndex.exorciseIndex(status);
        checkIndex.close();
        //测试修复完毕之后索引是否能够打开
        IndexWriter indexWriter = new IndexWriter(FSDirectory.open(Paths.get(indexFilePath)), new IndexWriterConfig(new
                StandardAnalyzer()));
        System.out.println(indexWriter.isOpen());
        indexWriter.close();
    }
}
```
如果索引完好，输出如下信息：
>Segments file=segments_2 numSegments=2 version=6.0.0 id=2jlug2dgsc4tmgkf5rck1xgm4
  1 of 2: name=_0 maxDoc=1
    version=6.0.0
    id=2jlug2dgsc4tmgkf5rck1xgm1
    codec=Lucene60
    compound=true
    numFiles=3
    size (MB)=0.002
    diagnostics = {java.runtime.version=1.8.0_45-b15, java.vendor=Oracle Corporation, java.version=1.8.0_45, java.vm.version=25.45-b02, lucene.version=6.0.0, os=Windows 7, os.arch=amd64, os.version=6.1, source=flush, timestamp=1464159740092}
    no deletions
    test: open reader.........OK [took 0.051 sec]
    test: check integrity.....OK [took 0.000 sec]
    test: check live docs.....OK [took 0.000 sec]
    test: field infos.........OK [2 fields] [took 0.000 sec]
    test: field norms.........OK [0 fields] [took 0.000 sec]
    test: terms, freq, prox...OK [1 terms; 1 terms/docs pairs; 0 tokens] [took 0.007 sec]
    test: stored fields.......OK [2 total field count; avg 2.0 fields per doc] [took 0.008 sec]
    test: term vectors........OK [0 total term vector count; avg 0.0 term/freq vector fields per doc] [took 0.000 sec]
    test: docvalues...........OK [0 docvalues fields; 0 BINARY; 0 NUMERIC; 0 SORTED; 0 SORTED_NUMERIC; 0 SORTED_SET] [took 0.000 sec]
    test: points..............OK [1 fields, 1 points] [took 0.001 sec]
  2 of 2: name=_1 maxDoc=1001
    version=6.0.0
    id=2jlug2dgsc4tmgkf5rck1xgm3
    codec=Lucene60
    compound=true
    numFiles=4
    size (MB)=0.023
    diagnostics = {java.runtime.version=1.8.0_45-b15, java.vendor=Oracle Corporation, java.version=1.8.0_45, java.vm.version=25.45-b02, lucene.version=6.0.0, os=Windows 7, os.arch=amd64, os.version=6.1, source=flush, timestamp=1464159740329}
    has deletions [delGen=1]
    test: open reader.........OK [took 0.003 sec]
    test: check integrity.....OK [took 0.000 sec]
    test: check live docs.....OK [1000 deleted docs] [took 0.000 sec]
    test: field infos.........OK [3 fields] [took 0.000 sec]
    test: field norms.........OK [0 fields] [took 0.000 sec]
    test: terms, freq, prox...OK [1 terms; 1 terms/docs pairs; 0 tokens] [took 0.008 sec]
    test: stored fields.......OK [2 total field count; avg 2.0 fields per doc] [took 0.012 sec]
    test: term vectors........OK [0 total term vector count; avg 0.0 term/freq vector fields per doc] [took 0.000 sec]
    test: docvalues...........OK [0 docvalues fields; 0 BINARY; 0 NUMERIC; 0 SORTED; 0 SORTED_NUMERIC; 0 SORTED_SET] [took 0.000 sec]
    test: points..............OK [1 fields, 1001 points] [took 0.001 sec]
No problems were detected with this index.
Took 0.159 sec total.
Check Index successfully！

在破坏索引之后（删除了一个cfe文件），再次运行输出
>Segments file=segments_2 numSegments=2 version=6.0.0 id=2jlug2dgsc4tmgkf5rck1xgm4
  1 of 2: name=_0 maxDoc=1
    version=6.0.0
    id=2jlug2dgsc4tmgkf5rck1xgm1
    codec=Lucene60
    compound=true
    numFiles=3
FAILED
    WARNING: exorciseIndex() would remove reference to this segment; full exception:
java.nio.file.NoSuchFileException: D:\index_test\_0.cfe
    at sun.nio.fs.WindowsException.translateToIOException(WindowsException.java:79)
...
...
    at java.lang.reflect.Method.invoke(Method.java:497)
    at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
  2 of 2: name=_1 maxDoc=1001
    version=6.0.0
    id=2jlug2dgsc4tmgkf5rck1xgm3
    codec=Lucene60
    compound=true
    numFiles=4
    size (MB)=0.023
    diagnostics = {java.runtime.version=1.8.0_45-b15, java.vendor=Oracle Corporation, java.version=1.8.0_45, java.vm.version=25.45-b02, lucene.version=6.0.0, os=Windows 7, os.arch=amd64, os.version=6.1, source=flush, timestamp=1464159740329}
    has deletions [delGen=1]
    test: open reader.........OK [took 0.059 sec]
    test: check integrity.....OK [took 0.000 sec]
    test: check live docs.....OK [1000 deleted docs] [took 0.001 sec]
    test: field infos.........OK [3 fields] [took 0.000 sec]
    test: field norms.........OK [0 fields] [took 0.000 sec]
    test: terms, freq, prox...OK [1 terms; 1 terms/docs pairs; 0 tokens] [took 0.013 sec]
    test: stored fields.......OK [2 total field count; avg 2.0 fields per doc] [took 0.016 sec]
    test: term vectors........OK [0 total term vector count; avg 0.0 term/freq vector fields per doc] [took 0.000 sec]
    test: docvalues...........OK [0 docvalues fields; 0 BINARY; 0 NUMERIC; 0 SORTED; 0 SORTED_NUMERIC; 0 SORTED_SET] [took 0.000 sec]
    test: points..............OK [1 fields, 1001 points] [took 0.002 sec]
WARNING: 1 broken segments (containing 1 documents) detected
Took 0.165 sec total.
Starting repair index files...
true