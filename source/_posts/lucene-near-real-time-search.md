---
title: Lucene 近实时搜索
date: 2017-12-12 22:35:56
tags: [Lucene]
categories: Programming Notes

---
### Lucene 事务
有过数据库经验的人都知道ACID特性，原子性（atomicity，或称不可分割性）、一致性（consistency）、隔离性（isolation，又称独立性）、持久性（durability）。由于隔离性的存在，对于新的变更包括添加、修改、删除，如果不进行 commit 的话，那么在读端是无法看到数据的变化的，在这里简单的介绍下 Lucene 中的事务，即ACID。

#### 原子性
当你在一次 IndexWriter 的 session 中做操作（增加，删除文档），然后 commit，要么你的所有的操作修改都是可见的（commit 成功），要么所有的操作修改都不可见（commit 失败），绝不会处于某种中间状态。有些方法有它自身的原子操作：如果你调用 updateDocument 方法，其内在实现是先删除后添加文档，即使你打开了一个近实时（NRT）reader 或者使用另一个线程做 commit，绝不会出现只有删除而没有添加的情况。与此类似，如果使用 addDocuments 方法添加一组文档，对于任何 reader 而言，要么所有的文档可见，要么所有文档不可见。

#### 一致性
如果计算机或者 OS 崩溃，或者 JVM 挂掉或被杀死，亦或是电源被拔掉了，你的索引都会保持完好。注意，像 RAM 故障，CPU 位翻转或者文件系统损坏之类的问题，还是容易造成索引破坏的。

#### 隔离性
当 IndexWriter 正在做更改的时候，所有更改都不会对当前搜索该索引的 IndexReader 可见，直到你 commit 或者打开了一个新的 NRT reader。一次只能有一个 IndexWriter 实例对索引进行更改。

#### 持久性
一旦 commit 操作返回，所有变更都会被写入到持久化存储。如果计算机或者 OS 崩溃，或者 JVM 挂掉或被杀死，亦或是电源被拔掉了，所有的变更都已在索引中保存。

### Lucene 近实时搜索
如果对数据没有实时性的要求，那么完全不用关心 Lucene 的近实时搜索。但是由于隔离性的存在，我们知道，对于新的数据，如果写端不 commit 的话，那么搜索端则是不可见的；还有另外一种情况，就是在搜索端可以持有 IndexWriter 的情况下，利用 IndexWriter 打开 reader，则可以保证 reader 能够看到最新的数据，包括还未提交的数据。同样一旦 reader 打开索引之后，对于写端新的未提交数据同样不可见。

除此之外 Lucene 与数据库还稍有不同，就是 Lucene 使用 IndexReader 打开索引的时候，相当于是对当前索引做了一次快照（Snapshot），即便写端进行了 commit 操作，如果 IndexReader 不重新打开的话，新提交的内容对搜索端依然是不可见的。那么要实现 Lucene 的近实时搜索，关键有两个方面，一是写端定期进行 commit 操作，二是搜索端定期进行 reopen 操作。

#### 搜索端持有 IndexWriter
对于大规模的搜索应用，一般都是读写分离的，而且由于 Lucene 在同一时刻，仅允许有一个写端存在，但可以有多个读端同时存在，所以这种架构在生产环境其实很少使用，因为所有的读端基本上都是和写端隔离甚至分别部署在不同的应用之中，不过这个知识在某些场景依然很有用，同时了解这个知识可以帮助你更加深入的理解 Lucene。

##### 由自己负责重新打开
```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.MatchAllDocsQuery;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

public class NearRealTimeSearchDemo {
    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                //即便没有commit，依然能看到Doc1
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        indexWriter.commit();
        count = indexSearcher.count(new MatchAllDocsQuery());
        TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
        for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
            //即便后来commit了，依然无法看到Doc2，因为没有重新打开IndexSearcher
            System.out.println(indexSearcher.doc(scoreDoc.doc));
        }
        System.out.println("=================================");
        indexSearcher.getIndexReader().close();
        indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
        count = indexSearcher.count(new MatchAllDocsQuery());
        topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
        for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
            //重新打开IndexSearcher，可以看到Doc1和Doc2
            System.out.println(indexSearcher.doc(scoreDoc.doc));
        }
        System.out.println("=================================");
    }
}
```
输出结果
```text
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
=================================
```

##### 使用ControlledRealTimeReopenThread定时重新打开
Lucene 提供了 ControlledRealTimeReopenThread 线程工具类来负责周期性的打开 ReferenceManager（调用ReferenceManager.maybeRefresh）。该线程类控制打开间隔比较灵活，当有外部用户在等待指定的 generation 时就按最小时间间隔等待，如果没有用户着急获取最新的Searcher，则等待最大时间间隔后再打开。其构造函数如下：

ControlledRealTimeReopenThread(TrackingIndexWriter writer, ReferenceManager<T> manager,
double targetMaxStaleSec, double targetMinStaleSec)//单位秒

如何判断是否有人等待？

在调用 addDocument 的时候，IndexWriter 为每次更新索引的操作赋予一个标记（generation，代数），递增变化。用户使用 ControlledRealTimeReopenThread.waitForGeneration 告诉其期望获得更新代数，ControlledRealTimeReopenThread 记录了当前已打开的代数，当期望更新代数大于已打开代数时，就表示有用户期望获得最新的 Searcher。
```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.*;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.util.concurrent.TimeUnit;

public class NearRealTimeSearchDemo {
    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        indexWriter.commit();
        SearcherManager searcherManager = new SearcherManager(indexWriter, null);
        //当没有调用者等待指定的generation的时候，必须要重新打开时间间隔5s，言外之意，如果有调用者在等待指定的generation，则只需等0.25s
        //防止不断的重新打开，严重消耗系统性能，设置最小重新打开时间间隔0.25s
        ControlledRealTimeReopenThread<IndexSearcher> controlledRealTimeReopenThread = new ControlledRealTimeReopenThread<>(indexWriter,
                searcherManager, 5, 0.25);
        //设置为后台线程
        controlledRealTimeReopenThread.setDaemon(true);
        controlledRealTimeReopenThread.setName("controlled reopen thread");
        controlledRealTimeReopenThread.start();
        int count = 0;
        IndexSearcher indexSearcher = searcherManager.acquire();
        try {
            //只能看到Doc1
            count = indexSearcher.count(new MatchAllDocsQuery());
            if (count > 0) {
                TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
                for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                    System.out.println(indexSearcher.doc(scoreDoc.doc));
                }
            }
            System.out.println("=================================");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            searcherManager.release(indexSearcher);
        }
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        try {
            TimeUnit.SECONDS.sleep(6);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //休息6s之后，即使没有commit，依然可以搜索到Doc2，因为ControlledRealTimeReopenThread会刷新SearchManager
        indexSearcher = searcherManager.acquire();
        try {
            count = indexSearcher.count(new MatchAllDocsQuery());
            if (count > 0) {
                TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
                for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                    System.out.println(indexSearcher.doc(scoreDoc.doc));
                }
            }
            System.out.println("=================================");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            searcherManager.release(indexSearcher);
        }
        document = new Document();
        document.add(new TextField("title", "Doc3", Field.Store.YES));
        document.add(new IntPoint("ID", 3));
        long generation = indexWriter.addDocument(document);
        try {
            //当有调用者等待某个generation的时候，只需要0.25s即可重新打开
            controlledRealTimeReopenThread.waitForGeneration(generation);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        indexSearcher = searcherManager.acquire();
        try {
            count = indexSearcher.count(new MatchAllDocsQuery());
            if (count > 0) {
                TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
                for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                    System.out.println(indexSearcher.doc(scoreDoc.doc));
                }
            }
            System.out.println("=================================");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            searcherManager.release(indexSearcher);
        }
    }
}
```

输出结果
```text
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
Document<stored,indexed,tokenized<title:Doc3>>
=================================
```

#### 搜索端不持有 IndexWriter
在实际的工作中，这种模式才是最常见的现象，由于索引可以有多个用户同时搜索，那么只要每一个搜索端都不持有 IndexWriter，就可以实现一个简单的分布式搜索服务。由前面所知，这种情况下，只有当写端进行 commit 操作之后，搜索端重新打开 reader 才能搜索到最新的数据。

##### 使用DirectoryReader刷新
在示例代码中虽然 IndexWriter 和 IndexSearcher 写在同一个类中，但是我们并不从 IndexWriter 打开索引，而是通过 Directory 打开索引，这里仅是简单示例，所以实际开发中，这个类中完全没有 IndexWriter 也是可以的。

想要看到新的结果就需要重新打开一个 IndexReader，DirectoryReader 提供了 openIfChanged(DirectoryReader oldReader) 函数，只有索引有变化时才返回新的 reader（不是完全打开一个 new reader，会复用 old reader 的一些资源，并入新索引，降低一些开销）， 否则返回 null。

```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.MatchAllDocsQuery;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

public class NearRealTimeSearchDemo {
    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        indexWriter.commit();
        DirectoryReader directoryReader = DirectoryReader.open(ramDirectory);
        IndexSearcher indexSearcher = new IndexSearcher(directoryReader);
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        //即使commit，搜索依然不可见，需要重新打开reader
        indexWriter.commit();
        //只能看到Doc1
        int count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        //如果发现有新数据更新，则会返回一个新的reader
        DirectoryReader newReader = DirectoryReader.openIfChanged(directoryReader);
        if (newReader != null) {
            indexSearcher = new IndexSearcher(newReader);
            directoryReader.close();
        }
        count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
    }
}
```
输出结果
```text
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
=================================
```

##### 使用SearcherManager刷新
在实际应用中，会并行的进行搜索、建索引、打开新 reader、关闭老 reader，操作比较复杂还有线程安全问题。为了简化使用流程，Lucene 提供了 SearcherManager extends ReferenceManager<IndexSearcher> 管理 IndexReader 的重建和关闭，保证了线程安全，封装了 IndexSearcher 的生成。

SearcherManager的主要职责如下，主要提供如下三个接口
- acquire：获取当前已打开的最新 IndexSearcher，同时将对应的 IndexReader 引用计数加1
- release：释放之前通过 acquire 方法获取的引用，本质调的是 `IndexSearcher.getIndexReader().decRef();`，当一个 IndexReader 的引用计数为0时，会将之关闭同时释放其持有的资源
- maybeRefresh：尝试打开新的 IndexReader，本质调用的是 DirectoryReader.openIfChanged() 方法，如果多个线程同时调用该方法，那么只有第一个线程会去尝试刷新，后来的或者说随后的线程看到有其它线程在处理刷新，那么会立即返回；注意这意味着，如果有一个线程在处理刷新操作，那么后续的线程会立即返回而不是等待它刷新完成，所以后续调用该方法的线程可能是已经刷新了的或者是没有任何改变的（没有任何改变意味着通过 acquire 获取的 IndexSearcher 无法搜索到最新的数据）
- maybeRefreshBlocking：功能同 maybeRefresh()，但是不像 maybeRefresh()，如果有一个线程正在刷新，后续的线程将阻塞，直到前面的线程刷新完成，然后后续的线程再继续刷新；这非常有用对于想要保证下一次 acquire() 的调用将返回一个刷新的实例，否则考虑使用 maybeRefresh()

SearcherManager 的一个注意点是，如果调用了 acquire，那么一定要调用 release，在该方法内部会通过 IndexSearcher 获取IndexReader，然后将 IndexReader 的引用计数减1，当这个引用计数为0的时候，这个 IndexReader 就会被关闭，从而可以释放资源。

```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.*;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

public class NearRealTimeSearchDemo {
    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        indexWriter.commit();
        SearcherManager searcherManager = new SearcherManager(ramDirectory, null);
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        //即使commit，搜索依然不可见，需要重新打开reader
        indexWriter.commit();
        //只能看到Doc1
        IndexSearcher indexSearcher = searcherManager.acquire();
        int count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        //最好放到finally语句块中
        searcherManager.release(indexSearcher);
        if (!searcherManager.isSearcherCurrent()) {
            //说明有新数据，需要刷新
            searcherManager.maybeRefresh();
        }
        indexSearcher = searcherManager.acquire();
        count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        searcherManager.release(indexSearcher);
    }
}
```

输出结果
```text
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
=================================
```
##### 使用定时线程执行刷新
```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.*;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class NearRealTimeSearchDemo {
    private static SearcherManager searcherManager;

    public static void main(String[] args) throws IOException {
        ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();
        executorService.scheduleWithFixedDelay(new RefreshThread(), 1, 1, TimeUnit.SECONDS);
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        indexWriter.commit();
        searcherManager = new SearcherManager(ramDirectory, null);
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        //即使commit，搜索依然不可见，需要重新打开reader
        indexWriter.commit();
        //只能看到Doc1
        IndexSearcher indexSearcher = searcherManager.acquire();
        int count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        //最好放到finally语句块中
        searcherManager.release(indexSearcher);
        //休息2s中，定时线程应该已经刷新了
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        indexSearcher = searcherManager.acquire();
        count = indexSearcher.count(new MatchAllDocsQuery());
        if (count > 0) {
            TopDocs topDocs = indexSearcher.search(new MatchAllDocsQuery(), count);
            for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
                System.out.println(indexSearcher.doc(scoreDoc.doc));
            }
        }
        System.out.println("=================================");
        searcherManager.release(indexSearcher);
    }

    static class RefreshThread implements Runnable {
        @Override
        public void run() {
            try {
                searcherManager.maybeRefresh();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

输出结果

```text
Document<stored,indexed,tokenized<title:Doc1>>
=================================
Document<stored,indexed,tokenized<title:Doc1>>
Document<stored,indexed,tokenized<title:Doc2>>
=================================
```

### SearcherLifetimeManager管理Search Session
管理 IndexSearcher 的生命周期主要是用在分页的时候，例如如果不进行管理的话，那么在用户提交了更改到索引中，正好此时后台线程开始刷新 IndexSearcher，如果提交的内容和用户的搜索不相关的话还好，如果提交的内容是和用户搜索结果相关的，那么新增加的内容可能会影响结果排序，从而在分页的时候可能会使用户点击翻页却看到相同的内容。

为了保证一个良好的用户体验，需要对用户在一个搜索会话期间分页的时候使用同样一个 IndexSearcher，这样在分页期间，可以保证搜索结果的排序是稳定不变的，从而用户也不会再次看到同样的内容。SearcherLifetimeManager 主要有如下几个方法

```java
public IndexSearcher acquire(long version) //根据version获取之前记录的IndexSearcher，引用计数加1
public void release(IndexSearcher s) //释放之前通过acquire获取的IndexSearcher，引用计数减1
public long record(IndexSearcher searcher) //把当前正在使用的IndexSearcher放入内部的ConcurrentHashMap，并返回一个token，之后可以通过该token再次获取该IndexSearcher
public synchronized void prune(Pruner pruner) //按指示的age关闭IndexSearcher，并将它们从ConcurrentHashMap中移除
```

使用示例

```java
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntPoint;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.SearcherLifetimeManager;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

public class SearcherLifetimeManagerDemo {

    public static void main(String[] args) throws IOException {
        SearcherLifetimeManager searcherLifetimeManager = new SearcherLifetimeManager();
        RAMDirectory ramDirectory = new RAMDirectory();
        Document document = new Document();
        document.add(new TextField("title", "Doc1", Field.Store.YES));
        document.add(new IntPoint("ID", 1));
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
        indexWriter.addDocument(document);
        indexWriter.commit();
        document = new Document();
        document.add(new TextField("title", "Doc2", Field.Store.YES));
        document.add(new IntPoint("ID", 2));
        indexWriter.addDocument(document);
        indexWriter.commit();
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        //记录当前的searcher，保存token，当有后续的搜索请求到来，例如用户翻页，那么用这个token去获取对应的那个searcher
        long record = searcherLifetimeManager.record(indexSearcher);
        indexSearcher = searcherLifetimeManager.acquire(record);
        if (indexSearcher != null) {
            // Searcher is still here
            try {
                // do searching...
            } finally {
                searcherLifetimeManager.release(indexSearcher);
                // Do not use searcher after this!
                indexSearcher = null;
            }
        } else {
            // Searcher was pruned -- notify user session timed out, or, pull fresh searcher again
        }
        //由于保留许多的searcher是非常耗系统资源的，包括打开发files和RAM，所以最好在一个单独的线程中，定期的重新打开searcher和定时的去清理旧的searcher
        //丢弃所有比指定的时间都老的searcher
        searcherLifetimeManager.prune(new SearcherLifetimeManager.PruneByAge(600.0));
    }
}
```
**Notes**：本文代码基于 Lucene 7.0.0 开发
