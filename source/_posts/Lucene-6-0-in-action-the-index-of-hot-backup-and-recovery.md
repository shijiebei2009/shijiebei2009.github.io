title: Lucene 6.0 实战-索引热备份及恢复
date: 2016-06-24 23:24:43
tags: [Lucene]
categories: Programming Notes

---

### 索引备份的几个关键问题
- 最简单的备份方式是关闭IndexWriter，然后逐一拷贝索引文件，但是如果索引比较大，那么这种备份操作会持续较长时间，而在备份期间，程序无法对索引文件进行修改，很多搜索程序是不能接受索引操作期间如此长时间停顿的
- 那么不关闭IndexWriter又如何呢？这样也不行，因为在拷贝索引期间，如果索引文件发生变化，会导致备份的索引文件损坏
- 另外一个问题就是如果原索引文件损坏的话，再备份它也毫无意义，所以一定要备份的是最后一次成功commit之后的索引文件
- 每次在备份之前，如果程序将要覆盖上一个备份，需要先删除备份中未出现在当前快照中的文件，因为这些文件已经不会被当前索引引用了；如果每次都更改备份路径的话，那么就直接拷贝即可

### 索引热备份
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
                    File to = new File(backupIndexDir + File.separator