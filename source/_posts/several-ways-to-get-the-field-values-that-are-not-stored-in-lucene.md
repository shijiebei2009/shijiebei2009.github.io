---
title: Lucene 中获取没有存储的字段值的几种方法
date: 2017-10-10 22:31:02
tags: [Lucene]
categories: Programming Notes

---

一般来说，如果想要从Lucene索引中获取Field的值，那么需要在索引阶段设置Field.Store.YES才可以，然后在搜索阶段得到TopDocs对象之后，用它去获取ScoreDoc再取出Document，使用Document获取存储在索引中的值。但是我们都知道，存储字段是需要硬盘空间的，如果想要追求极致的存储空间并且获取Field的值，那么在不存储的情况下，如何获取呢？其实仔细思索一下，在我们只索引不存储的情况下，Lucene依然可以判断搜索是否命中，这说明在Lucene索引中依然存有一份Field的值，这样在搜索阶段才能判断是否匹配。本文就是探讨在这种情形下，使用Lucene的核心包获取没有存储的Field的值的几种方法，如果你还有其它不同的方法请留言。

- **testGetFieldByStore** 演示存储Field值时如何获取
- **testGetFieldByTerms** 演示通过Terms获取没有存储的Field值
- **testGetFieldByFieldDocWithSorted** 演示通过FieldDoc获取没有存储的值
- **testGetFieldByTermVector** 演示通过TermVector获取没有存储的值
- **testGetFieldByTermVectors** 演示通过TermVectors获取没有存储的值

这里补充一下，在lucene-suggest包中，有LuceneDictionary类，通过该类的getEntryIterator方法也能获取没有存储的Field的值，不过其本质和通过Terms获取方式一样，在此不再列举。源码示例如下
```java
import org.apache.lucene.analysis.core.WhitespaceAnalyzer;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.search.*;
import org.apache.lucene.store.RAMDirectory;
import org.apache.lucene.util.BytesRef;
import org.junit.Test;

import java.io.IOException;

import static org.apache.lucene.search.SortField.Type.STRING;

/**
 * <p>
 * Created by wangxu on 2017/10/10 17:33.
 * </p>
 * <p>
 * Description: Lucene 6.5.0
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class GetNonStoredFieldDemo {
    private RAMDirectory ramDirectory = new RAMDirectory();
    private IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new WhitespaceAnalyzer()));

    public GetNonStoredFieldDemo() throws IOException {
    }

    @Test
    public void testGetFieldByStore() throws IOException {
        initIndexForStore();
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        TopDocs search = indexSearcher.search(new MatchAllDocsQuery(), count);
        ScoreDoc[] scoreDocs = search.scoreDocs;
        for (ScoreDoc scoreDoc : scoreDocs) {
            Document doc = indexSearcher.doc(scoreDoc.doc);
            System.out.println(doc.get("IDX") + "=>" + doc.get("title"));
        }
        ramDirectory.close();
    }

    @Test
    public void testGetFieldByTerms() throws IOException {
        initIndexForTerms();
        Fields fields = MultiFields.getFields(DirectoryReader.open(ramDirectory));
        Terms idx = fields.terms("IDX");
        Terms title = fields.terms("title");
        //or you can use like this
        //TermsEnum idxIter = MultiFields.getTerms(DirectoryReader.open(ramDirectory), "IDX").iterator();
        TermsEnum idxIter = idx.iterator();
        TermsEnum titleIter = title.iterator();
        BytesRef bytesRef;
        while ((bytesRef = idxIter.next()) != null) {
            System.out.println(bytesRef.utf8ToString() + "=>" + titleIter.next().utf8ToString());
        }
        ramDirectory.close();
    }

    @Test
    public void testGetFieldByFieldDocWithSorted() throws IOException {
        initIndexForFieldDocWithSorted();
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        //must use method which returns TopFieldDocs
        TopFieldDocs search = indexSearcher.search(new MatchAllDocsQuery(), count, new Sort(new SortField("IDX", STRING)));
        ScoreDoc[] scoreDocs = search.scoreDocs;
        for (ScoreDoc scoreDoc : scoreDocs) {
            FieldDoc fieldDoc = (FieldDoc) scoreDoc;
            Object[] fields = fieldDoc.fields;
            if (fields[0] instanceof BytesRef) {
                BytesRef temp = (BytesRef) fields[0];
                System.out.println(temp.utf8ToString() + "=>" + indexSearcher.doc(scoreDoc.doc).get("title"));
            }
        }
        ramDirectory.close();
    }

    @Test
    public void testGetFieldByTermVector() throws IOException {
        initIndexForTermVector();
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        TopDocs search = indexSearcher.search(new MatchAllDocsQuery(), count);
        ScoreDoc[] scoreDocs = search.scoreDocs;
        for (ScoreDoc scoreDoc : scoreDocs) {
            int doc = scoreDoc.doc;
            Terms idx = indexSearcher.getIndexReader().getTermVector(doc, "IDX");
            TermsEnum iterator = idx.iterator();
            BytesRef bytesRef;
            while ((bytesRef = iterator.next()) != null) {
                System.out.println(bytesRef.utf8ToString() + "=>" + indexSearcher.doc(doc).get("title"));
            }
        }
        ramDirectory.close();
    }

    @Test
    public void testGetFieldByTermVectors() throws IOException {
        initIndexForTermVector();
        IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
        int count = indexSearcher.count(new MatchAllDocsQuery());
        TopDocs search = indexSearcher.search(new MatchAllDocsQuery(), count);
        ScoreDoc[] scoreDocs = search.scoreDocs;
        for (ScoreDoc scoreDoc : scoreDocs) {
            int doc = scoreDoc.doc;
            Fields termVectors = indexSearcher.getIndexReader().getTermVectors(doc);
            Terms idx = termVectors.terms("IDX");
            TermsEnum iterator = idx.iterator();
            BytesRef bytesRef;
            while ((bytesRef = iterator.next()) != null) {
                System.out.println(bytesRef.utf8ToString() + "=>" + indexSearcher.doc(doc).get("title"));
            }
        }
        ramDirectory.close();
    }

    private void initIndexForStore() throws IOException {
        Document document = new Document();
        document.add(new StringField("IDX", "TEST01", Field.Store.YES));
        document.add(new StringField("title", "TITLE01", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST02", Field.Store.YES));
        document.add(new StringField("title", "TITLE02", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST03", Field.Store.YES));
        document.add(new StringField("title", "TITLE03", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST04", Field.Store.YES));
        document.add(new StringField("title", "TITLE04", Field.Store.YES));
        indexWriter.addDocument(document);
        indexWriter.close();
    }

    private void initIndexForTerms() throws IOException {
        Document document = new Document();
        document.add(new StringField("IDX", "TEST01", Field.Store.NO));
        document.add(new StringField("title", "TITLE01", Field.Store.NO));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST02", Field.Store.NO));
        document.add(new StringField("title", "TITLE02", Field.Store.NO));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST03", Field.Store.NO));
        document.add(new StringField("title", "TITLE03", Field.Store.NO));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new StringField("IDX", "TEST04", Field.Store.NO));
        document.add(new StringField("title", "TITLE04", Field.Store.NO));
        indexWriter.addDocument(document);
        indexWriter.close();
    }

    private void initIndexForTermVector() throws IOException {
        FieldType fieldType = new FieldType();
        fieldType.setStoreTermVectors(true);
        fieldType.setIndexOptions(IndexOptions.DOCS);
        Document document = new Document();
        document.add(new Field("IDX", "TEST01", fieldType));
        document.add(new StringField("title", "TITLE01", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new Field("IDX", "TEST02", fieldType));
        document.add(new StringField("title", "TITLE02", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new Field("IDX", "TEST03", fieldType));
        document.add(new StringField("title", "TITLE03", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new Field("IDX", "TEST04", fieldType));
        document.add(new StringField("title", "TITLE04", Field.Store.YES));
        indexWriter.addDocument(document);
        indexWriter.close();
    }

    private void initIndexForFieldDocWithSorted() throws IOException {
        Document document = new Document();
        document.add(new SortedDocValuesField("IDX", new BytesRef("TEST01")));
        document.add(new StringField("title", "TITLE01", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new SortedDocValuesField("IDX", new BytesRef("TEST02")));
        document.add(new StringField("title", "TITLE02", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new SortedDocValuesField("IDX", new BytesRef("TEST03")));
        document.add(new StringField("title", "TITLE03", Field.Store.YES));
        indexWriter.addDocument(document);

        document = new Document();
        document.add(new SortedDocValuesField("IDX", new BytesRef("TEST04")));
        document.add(new StringField("title", "TITLE04", Field.Store.YES));
        indexWriter.addDocument(document);
        indexWriter.close();
    }

}
```