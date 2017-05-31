---
title: 使用DocumentStoredFieldVisitor提高Lucene检索速度
tags: [Lucene]
categories: Programming Notes
date: 2017-05-31 21:42:38

---

#### FieldSelector
提高Lucene检索性能的方法有很多种，这里简单介绍一种常用且便捷可行的方法快速提高Lucene检索性能。在早期的Lucene版本中，使用**FieldSelector**来决定哪些Fields应该被加载，并以何种方式加载，但是在[LUCENE-3309](https://issues.apache.org/jira/browse/LUCENE-3309)中该接口被废弃，并且提出了新的替代接口**StoredFieldVisitor**。

#### FieldCache
另一种提高检索性能的方案是使用FieldCache来缓存Lucene的term values信息，不过该接口目前已被移至`org.apache.lucene.uninverting`包下，并且访问权限变成包级私有，也就是说，用户再也无法直接使用FieldCache了，该接口以后仅限于Lucene内部使用。FieldCache的主要作用是缓存用来排序field的值，Lucene会将需要排序的字段都读到内存中进行排序，所占内存大小和文档数量相关。其替代方案是使用DocValues类。其实深入一步，当你的Document只有一个Token的时候，FieldCache还可以被用来快速获取每个Document的field值，因为Lucene只做了反向索引，这种Document->field正向索引是极其耗时的，而FieldCache正好能解决这个问题。

由于这两个接口基本相当于被废弃，这里不再赘述，主要讲解目前实用的**StoredFieldVisitor**方案。

#### StoredFieldVisitor
StoredFieldVisitor是一个抽象类，它有一个唯一对外暴露的实现类DocumentStoredFieldVisitor，查看该实现类的Doc文档说明，可知其作用是支持加载所有的stored fields，或者通过Set集合指定请求的fields。

查看DocumentStoredFieldVisitor构造函数
```java
public DocumentStoredFieldVisitor() {
  this.fieldsToAdd = null;
}

public DocumentStoredFieldVisitor(Set<String> fieldsToAdd) {
  this.fieldsToAdd = fieldsToAdd;
}

/** Load only fields named in the provided fields. */
public DocumentStoredFieldVisitor(String... fields) {
  fieldsToAdd = new HashSet<>(fields.length);
  for(String field : fields) {
    fieldsToAdd.add(field);
  }
}
```
一个无参构造函数，一个接收Set参数的构造函数，还有一个接收可变参数的构造函数，而可变参数的构造函数中其实就是把可变参数加入Set集合，所以其原理和接收Set集合的构造函数是一样的。

讲了这么多，那么DocumentStoredFieldVisitor的使用场景是什么呢？当用户需要访问各个文档中的某个field的值时，使用IndexSearcher.doc(int docID)可以获得Document，然后再从Document中获得某个域值，当一个Document中field非常多的时候，这种访问速度比较慢，而且只能获得Stored域的值。这时候使用DocumentStoredFieldVisitor可以极大地提高访问速度。下面写个简单的测试代码来看看其性能差距。

```java
import com.google.common.base.Stopwatch;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.lucene.analysis.cjk.CJKAnalyzer;
import org.apache.lucene.document.*;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.concurrent.*;

/**
 * <p>
 * Created by wangxu on 2017/05/27 17:37.
 * </p>
 * <p>
 * Description: 基于Lucene 6.5.0实现
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class TestStoredFieldVisitor {
    static IndexSearcher indexSearcher;
    static Query query;

    public static void main(String[] args) throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
        IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new CJKAnalyzer()));
        for (int i = 0; i < 1000000; i++) {
            Document document = new Document();
            document.add(new LongPoint("ID", i));
            document.add(new StringField("title", i + "title", Field.Store.YES));
            document.add(new TextField("content", i + "content", Field.Store.YES));
            if (i % 2 == 0) {
                document.add(new StringField("sex", "male", Field.Store.YES));
                document.add(new TextField("tags", "The " + i + "th male!", Field.Store.YES));
            } else {
                document.add(new StringField("sex", "female", Field.Store.YES));
                document.add(new TextField("tags", "The " + i + "th female!", Field.Store.YES));
            }
            document.add(new TextField("hobbies", "I like playing the " + i + " toys!", Field.Store.YES));
            document.add(new StringField("testField1", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField2", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField3", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField4", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField5", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField6", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField7", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            document.add(new StringField("testField8", RandomStringUtils.randomAlphabetic(10), Field.Store.YES));
            indexWriter.addDocument(document);
        }
        indexWriter.commit();
        indexWriter.close();
        indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        query = LongPoint.newRangeQuery("ID", 0, Math.addExact(1000000, -1));
        int count = indexSearcher.count(query);
        long total = 0;
        for (int i = 0; i < 10; i++) {
            Future<Long> submit = Executors.newSingleThreadExecutor().submit(new Worker1(count));//average time cost by 10 times : 8024 ms
            //Future<Long> submit = Executors.newSingleThreadExecutor().submit(new Worker2(count));//average time cost by 10 times : 6507 ms
            try {
                total += submit.get();
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
        System.out.println("average time cost by 10 times : " + total / 10 + " ms");
    }

    static class Worker1 implements Callable<Long> {
        int count = 0;

        public Worker1(int count) {
            this.count = count;
        }

        @Override
        public Long call() throws Exception {
            Stopwatch started = Stopwatch.createStarted();
            List<String> titles = new ArrayList<>();
            if (count > 0) {
                TopDocs docs = indexSearcher.search(query, count);
                ScoreDoc[] scoreDocs = docs.scoreDocs;
                for (ScoreDoc scoreDoc : scoreDocs) {
                    Document doc = indexSearcher.doc(scoreDoc.doc);
                    titles.add(doc.get("title"));
                }
                System.out.println("No DocumentStoredFieldVisitor get title counts = " + titles.size());
            }
            long elapsed = started.elapsed(TimeUnit.MILLISECONDS);
            System.out.println("No DocumentStoredFieldVisitor: " + elapsed + " ms");
            return elapsed;
        }
    }

    static class Worker2 implements Callable<Long> {
        int count = 0;

        public Worker2(int count) {
            this.count = count;
        }

        @Override
        public Long call() throws Exception {
            Stopwatch started = Stopwatch.createStarted();
            Set<String> title = new HashSet<>();
            title.add("title");
            DocumentStoredFieldVisitor titleVisitor = new DocumentStoredFieldVisitor(title);
            if (count > 0) {
                TopDocs docs = indexSearcher.search(query, count);
                ScoreDoc[] scoreDocs = docs.scoreDocs;
                for (ScoreDoc scoreDoc : scoreDocs) {
                    indexSearcher.doc(scoreDoc.doc, titleVisitor);
                }
                Document document = titleVisitor.getDocument();
                System.out.println("With DocumentStoredFieldVisitor get title counts = " + document.getValues("title").length);
            }
            long elapsed = started.elapsed(TimeUnit.MILLISECONDS);
            System.out.println("With DocumentStoredFieldVisitor: " + elapsed + " ms");
            return elapsed;
        }
    }
}
```
在索引1000000个文档之后，每个文档添加14个不同类型的Field，分别运行Worker1和Worker2，进行十次的基于ID的范围查询，取十次结果的平均值，得到使用DocumentStoredFieldVisitor平均单次耗时6507 ms，不使用DocumentStoredFieldVisitor平均单次耗时8024 ms。可见性能提升还是很可观的，当然该测试并不权威，但是可以给出一个简单直观的比较。