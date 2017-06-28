---
title: 用RAMDirectory做缓存提高Lucene性能
date: 2017-06-28 22:02:21
tags: [Lucene]
categories: Programming Notes

---

RAMDirectory和FSDirectory都继承自BaseDirectory，而BaseDirectory继承自Directory，Directory是Lucene中设计的一个顶层抽象类，可以将其看做本地文件系统的一个目录。

RAMDirectory是基于内存实现的，具有较高的存储速度，但是受到内存大小的限制，而FSDirectory是基于文件系统实现的，针对不同的操作系统有不同的具体实现类，这些实现类无需用户操心，只需要调用FSDirectory.open(Path path)方法，它就会帮助我们选择最适合的子类，FSDirectory的瓶颈在于磁盘I/O。

如果机器内存足够大的话，那么组合使用RAMDirectory和FSDirectory将能够极大地提高Lucene的性能。组合使用两者的应用场景很多，不同的场景可以分别解决不同的需求，仅列举如下
- 批量索引，而无需检索的情况下，先把document存到RAMDirectory中，当达到一定数量之后，再把这些索引一次性加入FSDirectory里

```java
import org.apache.lucene.analysis.core.WhitespaceAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.StringField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.MatchAllDocsQuery;
import org.apache.lucene.store.FSDirectory;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.nio.file.Paths;

/**
 * <p>
 * Created by wangxu on 2017/06/19 15:31.
 * </p>
 * <p>
 * Description: 基于Lucene 6.5开发
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class RamFSDirectoryDemo {
    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        IndexWriter ramWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new WhitespaceAnalyzer()).setOpenMode(IndexWriterConfig.OpenMode.CREATE));

        FSDirectory fsDirectory = FSDirectory.open(Paths.get("F:/index_RAM_FS"));
        IndexWriter fsWriter = new IndexWriter(fsDirectory, new IndexWriterConfig(new WhitespaceAnalyzer()).setOpenMode(IndexWriterConfig.OpenMode
                .CREATE_OR_APPEND));

        //简单起见，这里的数量都比较少，例如索引10000个document，每100个document作为一批
        int tempCount = 0;
        for (int i = 0; i < 10000; i++) {
            Document document = new Document();
            document.add(new StringField("key", String.valueOf(i), Field.Store.YES));
            ramWriter.addDocument(document);
            tempCount++;
            if (tempCount % 100 == 0) {
                ramWriter.commit();
                ramWriter.close();
                fsWriter.addIndexes(ramDirectory);
                ramDirectory.close();
                fsWriter.commit();
                ramDirectory = new RAMDirectory();
                ramWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new WhitespaceAnalyzer()).setOpenMode(IndexWriterConfig.OpenMode.CREATE));
            }
        }
        //退出时确保RAMDirectory中内容都被加入本地
        if (ramWriter != null) {
            ramWriter.commit();
            ramWriter.close();
            fsWriter.addIndexes(ramDirectory);
            ramDirectory.close();
        }
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(fsWriter.getDirectory()));
        fsWriter.close();
        int count = indexSearcher.count(new MatchAllDocsQuery());
        System.out.println(count);
    }
}
```

- 索引的同时需要搜索，前提是内存总量大于索引文件总量，如果要求新加入的索引对搜索可见，即实时搜索，要怎么做呢？显然实时搜索需要writer和searcher共用同一份索引，同时要定时的将内存中索引备份到文件系统，否则机器一旦宕机，内存中所有的索引文件都将丢失。代码实现也很简单，在备份的时候，可以使用调度线程去进行备份操作，同时还不影响主线程继续接受索引请求；备份策略有两种：全量和增量，增量直接比较文件名，将新增文件拷贝到文件系统，同时删除已过时的索引文件即可。

```java
import org.apache.lucene.analysis.core.WhitespaceAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.StringField;
import org.apache.lucene.index.*;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.MatchAllDocsQuery;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.apache.lucene.store.IOContext;
import org.apache.lucene.store.RAMDirectory;

import java.io.File;
import java.io.IOException;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.*;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * <p>
 * Created by wangxu on 2017/06/19 15:31.
 * </p>
 * <p>
 * Description: 基于Lucene 6.5开发
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class RamFSDirectoryDemo {
    private static IndexWriter ramWriter;
    private static RAMDirectory ramDirectory;
    private static final Backup backup = new Backup();

    public static void main(String[] args) throws IOException {
        //将文件系统中索引文件加载到内存中
        FSDirectory fsDirectory = FSDirectory.open(Paths.get("F:/index_RAM_FS"));
        ramDirectory = new RAMDirectory(fsDirectory, IOContext.READONCE);
        fsDirectory.close();

        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new WhitespaceAnalyzer());
        indexWriterConfig.setIndexDeletionPolicy(new SnapshotDeletionPolicy(new KeepOnlyLastCommitDeletionPolicy()));
        ramWriter = new IndexWriter(ramDirectory, indexWriterConfig);

        //先添加一条记录
        Document document = new Document();
        document.add(new StringField("first", "first", Field.Store.YES));
        ramWriter.addDocument(document);
        ramWriter.commit();

        //定时备份线程
        ScheduledExecutorService backupThread = Executors.newSingleThreadScheduledExecutor();
        backupThread.scheduleAtFixedRate(backup, 0, 2, TimeUnit.SECONDS);

        //可以继续接受索引请求
        document = new Document();
        document.add(new StringField("key", "key", Field.Store.YES));
        ramWriter.addDocument(document);
        ramWriter.commit();

        //等待索引拷贝完成
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        //接受搜索请求，可实现实时搜索
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramWriter.getDirectory()));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        System.out.println("total hits: " + count);
        System.out.println("内存中索引文件：" + Arrays.asList(ramDirectory.listAll()));
        //查看备份索引是否完整，此处有一个注意点，如果需要在备份索引上打开searcher，那么在备份索引文件的时候需要先备份其它文件，最后再备份segments_N文件
        //因为开searcher的时候，会先加载segments_N文件，这种方式可以保证加载完segments_N文件之后，再加载其它文件一定成功

        fsDirectory = FSDirectory.open(Paths.get("F:/index_RAM_FS"));
        System.out.println("备份中索引文件：" + Arrays.asList(fsDirectory.listAll()));
        backupThread.shutdown();
    }

    static class Backup implements Runnable {
        public void run() {
            IndexWriterConfig config;
            SnapshotDeletionPolicy indexDeletionPolicy = null;
            IndexCommit snapshot = null;
            try {
                ramWriter.commit();
                config = (IndexWriterConfig) ramWriter.getConfig();
                indexDeletionPolicy = (SnapshotDeletionPolicy) config.getIndexDeletionPolicy();
                snapshot = indexDeletionPolicy.snapshot();
                config.setIndexCommit(snapshot);
                Collection<String> fileNames = snapshot.getFileNames();
                Path path = Paths.get("F:/index_RAM_FS");
                //全量增量任选一种即可
                boolean b = incrementalBackup(fileNames, path);
                //boolean b = fullBackup(fileNames, path);
                if (!b) {
                    System.err.println("Backup occurs error!");
                }
            } catch (IOException e) {
                e.printStackTrace();
                System.err.println(e.getMessage());
            } finally {
                try {
                    indexDeletionPolicy.release(snapshot);
                } catch (IOException e) {
                    e.printStackTrace();
                    System.err.println(e.getMessage());
                }
            }
        }

        private boolean fullBackup(Collection<String> fileNames, Path path) {
            Objects.requireNonNull(path);
            Directory to;
            try {
                to = FSDirectory.open(path);
                // 全量备份，直接清空拷贝
                for (File file : path.toFile().listFiles()) {
                    file.delete();
                }
                for (String fileName : fileNames) {
                    to.copyFrom(ramDirectory, fileName, fileName, IOContext.DEFAULT);
                }
                to.close();
            } catch (IOException e) {
                e.printStackTrace();
                return false;
            }
            return true;
        }

        private boolean incrementalBackup(Collection<String> fileNames, Path path) {
            // 增量备份，稍微复杂一些，比较文件名，将ramDirectory中新增索引拷贝过去，同时将ramDirectory中不存在的索引但是在path中存在的旧索引删除
            Objects.requireNonNull(path);
            //fileNames被IndexCommit引用，需要重新构造set集合，进行移除操作
            Set<String> files = new HashSet<>(fileNames);
            for (File file : path.toFile().listFiles()) {
                if (files.contains(file.getName())) {
                    //该索引已存在，则不拷贝
                    files.remove(file.getName());
                } else {
                    //删除已经过时的索引
                    file.delete();
                }
            }
            //拷贝全部新增索引
            try {
                Directory to = FSDirectory.open(path);
                for (String file : files) {
                    to.copyFrom(ramDirectory, file, file, IOContext.DEFAULT);
                }
                to.close();
            } catch (IOException e) {
                e.printStackTrace();
                return false;
            }
            return true;
        }
    }
}
```

- 如果内存不够大，不能够存放全部的本地文件系统索引，那么如何实现呢？这时候，需要两套IndexWriter，一套用来处理本地文件系统中的索引，另一套在内存中可以接受新的索引请求；对索引的操作无非就是新增、删除、更新这三种，下面分别论述

 - 删除索引： 对于删除操作比较简单，两个IndexWriter都执行一次即可，如果其中一个IndexWriter中不存在待删除的索引的话，那么对索引文件不会有任何影响；这时候又分两种情况，一是待删除的索引在内存中，那么删除-提交-重新打开索引，耗时很短；二是待删除索引在硬盘上，这时候删除-提交，到底要不要重新打开，需要视业务而定，因为硬盘上的索引通常是很大的，如果频繁删除频繁重新打开的话，是很耗性能的，而如果不重新打开，这时候删除的索引对searcher是不可见的，也就是说用户仍然可以搜索到已删除的索引，比较好的方式是用一个后台线程去定时的重新打开硬盘索引，然后更新searcher即可，但是这样对于删除操作无法做到准实时，会有一定的延时。
 - 新增索引：所有的新增操作都使用内存中的IndexWriter，然后提交-重新打开索引-更新searcher，一切都是在内存中操作，耗时极短，对于实时搜索是解决了，而且Lucene中已经提供了MultiReader可以组合多个子reader，很适合这种情况；但是当内存中索引尺寸达到一定大小之后，需要将其合并到文件系统中；在合并的时候还需要另外再开一个内存中的Temp Ram IndexWriter用来接受索引合并期间的索引新增操作。不过这种方式实现起来很复杂，需要处理很多问题，例如在内存索引和硬盘索引合并期间如果有更新删除操作怎么处理？在合并之后需要更新MultiReader，但是旧的MultiReader上面还挂有搜索请求怎么办？在新开一个Temp Ram IndexWriter之后，如何让MultiReader覆盖住它？因为在这个Temp Ram IndexWriter中必须先有索引，然后才能开Reader，进而才能更新MultiReader；若新增索引速度实在太快，在合并过程没有完成的时候，内存索引又满了，要怎么办？除此以外还有很多问题需要考虑。
 - 更新索引：更新其实可以看作是两步操作，先删除后新增，这两步可以借鉴上面的论述。更新操作是无法同时应用于两个IndexWriter的，因为在Lucene中更新的逻辑是这样的，如果存在则更新，如果不存在则新增，那么假设一个IndexWriter中存在，另一个不存在，如果两个都应用更新，那么最后的结果很简单就是存在两份一样的document了。
 - 越是简单的程序越健壮，这种方案实现起来很复杂，即使这么复杂的程序实现了，其健壮性仍然值得担忧，所以不是特别推荐使用这种方案。