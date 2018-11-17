---
title: Lucene 实现任意词搜索命中并返回位置信息
date: 2017-10-13 22:29:56
tags: [Lucene]
categories: Programming Notes

---
### 背景
如果把这个标题拆分成两个来讲，那么每一个都很好解决，下文会进行详述，而如果把这两者看做是与条件并加上其它限制，则实现起来比较困难，本文就是要探讨在需求繁多的情况下，如何优雅地实现。比如需求如下
- 保留标点符号，否则去掉标点的话，在标点两边的词可能会匹配上，比如“你好，小甜甜”，去掉标点切分是『你|好|小|甜|甜』，那么『好小』有可能会命中，而如果切分成『你|好|，|小|甜|甜』，则『好小』无法命中
- 只要包含搜索词，要求对任意搜索词均可命中
  - 比如“我爱你中国”，不同的分词工具会切分出不同的结果
  - 『我|爱|你|中国』或者『我爱你|中国』或者『我|爱|你|中|国』等，那么要求搜索“我爱”或者“爱你”或者“你中”等都要命中
- 需要获取命中词的positions信息
  - 还是以“我爱你中国”为例，如果搜“你中”，那么需要返回结果命中，并给出positions信息，例如start=2表示“你中”在原文本中是从第2个位置开始
- 可以设置slop
  - 什么是slop，简单来说slop是指两个项的位置之间允许的最大间隔距离
  - 为什么要设置slop呢？比如小黄文为了防止敏感词被屏蔽，会在敏感词中间加上干扰词，例如`性%=$虐待`，那么直接搜`性虐待`无法命中
  - 只要设置slop为3，即相当于搜`性[***]虐待`，这里面的`[***]`就代表slop为3，可以匹配任意三个字
- 不要存储Field的值
  - 如果可以存储的话，可以通过Document获取原文本，再用TokenStream分析该文本，使用QueryScore初始化TokenStream的分析结果，遍历每个token根据TokenScore的得分判断是否命中，若命中则输出位置信息或者起始偏移量即可
  - 但是存储Field的值是需要占用硬盘空间的，当需要索引海量的文本的时候，会导致索引体积非常大，搜索性能变差
  - 当然还可以通过将索引拆分成多份存储实现降低索引体积的目的，这也是一个方法，不过治标不治本
- 不要存储TermVectors
  - 同样地如果可以存储的话，可以通过TermVectors获取Terms再遍历TermsEnum，获取PostingsEnum得到positions信息，缺点在于只能实现单个字（Term）的搜索匹配
  - 但是存储TermVectors同样占用硬盘空间，为了缩小索引体积，不要存储
- 实现与逻辑，比如搜索“你中 & 我爱”表示两个词都要命中
- 实现或逻辑，比如搜索“你中 | 我爱”表示两个词至少要有一个命中

如果想要实现百分之百的任意词搜索命中，那么只能按字切分，因为没有任何分词工具能够保证切出来的词与搜索词是一致的。在上面说到为了达到匹配干扰词的目的，需要设置slop，但是会有一定的误判率，本来不该匹配的在设置slop之后也匹配上了。除了设置slop这种方式，还有一种方法，就是在索引阶段只保留汉字，其它的标点符号和干扰符号统统去掉，当然这也存在一定的误判率，而且获取的positions信息已经不是原文本中正确的positions信息了。两种方式的权衡与取舍可以根据业务需求而定，这两种方式都不会漏判，但是均会有误判。

### 简单需求的实现
如果并不要求满足上面所有的需求，而仅仅满足其中任何一个，那么实现起来都是非常简单的，以下代码均基于Lucene 5.5.0实现，示例如下。

#### 仅要求任意搜索词命中
```java
@org.junit.Test
public void testAnyMatch() throws IOException {
    RAMDirectory ramDirectory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    document.add(new TextField("content", "我爱你中国", Field.Store.NO));
    indexWriter.addDocument(document);
    indexWriter.commit();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
    PhraseQuery phraseQuery = new PhraseQuery.Builder().add(new Term("content", "你")).add(new Term("content", "中")).setSlop(0).build();
    TopDocs search = indexSearcher.search(phraseQuery, Integer.MAX_VALUE);
    System.out.println(search.totalHits);
    //OR search like this
    MultiPhraseQuery multiPhraseQuery = new MultiPhraseQuery();
    Term first = new Term("content", "你");
    Term second = new Term("content", "中");
    multiPhraseQuery.add(new Term[]{first, second});
    search = indexSearcher.search(multiPhraseQuery, Integer.MAX_VALUE);
    System.out.println(search.totalHits);
}
```

#### 可以存储Field的值
```java
@org.junit.Test
public void testStoreFieldMatch() throws IOException, InvalidTokenOffsetsException {
    RAMDirectory ramDirectory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    document.add(new TextField("content", "我爱你中国", Field.Store.YES));
    indexWriter.addDocument(document);
    indexWriter.commit();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
    PhraseQuery phraseQuery = new PhraseQuery.Builder().add(new Term("content", "你")).add(new Term("content", "中")).setSlop(0).build();
    TopDocs search = indexSearcher.search(phraseQuery, Integer.MAX_VALUE);
    ScoreDoc[] scoreDocs = search.scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        String content = indexSearcher.doc(scoreDoc.doc).get("content");
        TokenStream contentStream = new StandardAnalyzer().tokenStream("content", content);
        CharTermAttribute charTermAttribute = contentStream.addAttribute(CharTermAttribute.class);
        OffsetAttribute offsetAttribute = contentStream.addAttribute(OffsetAttribute.class);
        QueryScorer queryScorer = new QueryScorer(phraseQuery);
        queryScorer.setMaxDocCharsToAnalyze(Integer.MAX_VALUE);
        TokenStream init = queryScorer.init(contentStream);
        if (init != null) {
            contentStream = init;
        }
        contentStream.reset();
        queryScorer.startFragment(null);
        int startOffset, endOffset;
        for (boolean next = contentStream.incrementToken(); next && (offsetAttribute.startOffset() < Integer.MAX_VALUE); next = contentStream.incrementToken()) {
            startOffset = offsetAttribute.startOffset();
            endOffset = offsetAttribute.endOffset();
            if (startOffset > content.length() || endOffset > content.length()) {
                throw new InvalidTokenOffsetsException("Token " + charTermAttribute.toString() + " exceeds length of provided text sized " + content.length());
            }
            float res = queryScorer.getTokenScore();
            if (res > Float.valueOf(0) && startOffset <= endOffset) {
                System.out.println("hits: " + content.substring(startOffset, endOffset) + ", start: " + startOffset);
            }
        }
        contentStream.close();
    }
}
```

#### 可以存储TermVectors的值
```java
@org.junit.Test
public void testTermVectorsMatch() throws IOException, InvalidTokenOffsetsException {
    RAMDirectory ramDirectory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    FieldType fieldType = new FieldType();
    fieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS);
    fieldType.setStoreTermVectorPositions(true);
    fieldType.setStoreTermVectors(true);
    document.add(new Field("content", "我爱你中国", fieldType));
    indexWriter.addDocument(document);
    indexWriter.commit();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
    Term searchTerm = new Term("content", "中");
    PhraseQuery phraseQuery = new PhraseQuery.Builder().add(searchTerm).setSlop(0).build();
    TopDocs search = indexSearcher.search(phraseQuery, Integer.MAX_VALUE);
    ScoreDoc[] scoreDocs = search.scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        Terms content = indexSearcher.getIndexReader().getTermVector(scoreDoc.doc, "content");
        TermsEnum iterator = content.iterator();
        BytesRef bytesRef;
        while ((bytesRef = iterator.next()) != null) {
            PostingsEnum postings = iterator.postings(null, PostingsEnum.ALL);
            if (postings.nextDoc() != Spans.NO_MORE_DOCS) {
                for (int i = 0; i < postings.freq(); i++) {
                    if (searchTerm.text().equals(bytesRef.utf8ToString())) {
                        System.out.println("hits: " + bytesRef.utf8ToString() + ", start: " + postings.nextPosition());
                    }
                }
            }
        }
    }
}
```

### 复杂需求的实现
实现某一个简单的需求就不再举例了，下面要讲解如何实现复杂的需求，也就是说，要同时满足上面的需求列表，而不仅仅是只满足其中的某一条。首先需要解决的就是分词之后保留标点符号的问题，在Lucene中，我并没有找到原生的支持保留标点符号的Analyzer，于是只能自己造轮子了。

#### 保留标点符号的分词器
```java
import lombok.extern.log4j.Log4j2;
import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.Tokenizer;
import org.apache.lucene.analysis.core.LowerCaseFilter;
import org.apache.lucene.analysis.pattern.PatternTokenizer;

import java.util.regex.Pattern;

@Log4j2
public class ReservePunctuationAnalyzer extends Analyzer {
    public ReservePunctuationAnalyzer() {

    }

    @Override
    protected TokenStreamComponents createComponents(String fieldName) {
        final Tokenizer source;
        source = new PatternTokenizer(Pattern.compile(""), -1);
        TokenStream result = new LowerCaseFilter(source);
        return new TokenStreamComponents(source, result);
    }
}
```
分词测试如下
```java
@Test
public void test() throws IOException {
    String input = "你好，小甜甜。";
    TokenStream test = new ReservePunctuationAnalyzer().tokenStream("test", input);
    CharTermAttribute charTermAttribute = test.addAttribute(CharTermAttribute.class);
    OffsetAttribute offsetAttribute = test.addAttribute(OffsetAttribute.class);
    test.reset();
    while (test.incrementToken()) {
        System.out.println("token:[" + charTermAttribute + "], offset:[" + offsetAttribute.startOffset() + "]");
    }
    test.close();
}
```
分词结果输出如下
>token:[你], offset:[0]
token:[好], offset:[1]
token:[，], offset:[2]
token:[小], offset:[3]
token:[甜], offset:[4]
token:[甜], offset:[5]
token:[。], offset:[6]

#### 任意词搜索命中并返回positions信息
下面再来解决在不存储Field、不存储TermVectors的情况下，如何实现任意词搜索命中并返回positions信息，同时还可以设置slop的值。要实现这些功能就需要用到SpanQuery及其一系列的子类，先来看一张继承关系图，这些都是即将要使用到的类。
<div align=center>
[![SpanQuery And Subclasses](https://raw.githubusercontent.com/shijiebei2009/img/master/blog/spanquery_and_subclasses.jpg "SpanQuery And Subclasses")](https://raw.githubusercontent.com/shijiebei2009/img/master/blog/spanquery_and_subclasses.jpg "SpanQuery And Subclasses")
</div>

SpanQuery的doc注释很简单，就一句话“Base class for span-based queries”，基于跨度查询的基类。而真正具有实际作用的是其各个子类。

```java
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.FieldType;
import org.apache.lucene.document.LongField;
import org.apache.lucene.index.*;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.spans.SpanNearQuery;
import org.apache.lucene.search.spans.SpanTermQuery;
import org.apache.lucene.search.spans.SpanWeight;
import org.apache.lucene.search.spans.Spans;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.util.List;

import static org.apache.lucene.search.spans.SpanNearQuery.newOrderedNearQuery;

/**
 * <p>
 * Created by wangxu on 2017/10/13 14:29.
 * </p>
 * <p>
 * Description: TODO
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class SpanNearQueryDemo {
    @org.junit.Test
    public void test() throws IOException {
        String input = "现有的中文分词算法可分为三大类：基于字符串匹配的类基分词方法、基于理解的分词方法和基于统计的分词方法。";
        RAMDirectory ramDirectory = new RAMDirectory();
        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new ReservePunctuationAnalyzer());
        try (IndexWriter indexWriter = new IndexWriter(ramDirectory, indexWriterConfig)) {
            Document document = new Document();
            FieldType fieldType = new FieldType();
            fieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS);
            Field field = new Field("title", input, fieldType);
            LongField IDX = new LongField("IDX", 1, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法是很难很复杂滴";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 2, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法可以大幅度提升程序性能";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 3, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);
            indexWriter.commit();

            IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
            SpanTermQuery first = new SpanTermQuery(new Term("title", "类"));
            SpanTermQuery second = new SpanTermQuery(new Term("title", "基"));
            SpanNearQuery spanNearQuery = newOrderedNearQuery("title").addClause(first).addClause(second).build();
            SpanWeight weight = spanNearQuery.createWeight(indexSearcher, true);
            List<LeafReaderContext> leaves = indexSearcher.getIndexReader().getContext().leaves();
            for (LeafReaderContext leaf : leaves) {
                Spans spans = weight.getSpans(leaf, SpanWeight.Postings.POSITIONS);
                while (spans.nextDoc() != Spans.NO_MORE_DOCS) {
                    Document doc = leaf.reader().document(spans.docID());
                    while (spans.nextStartPosition() != Spans.NO_MORE_POSITIONS) {
                        System.out.println("doc id = " + spans.docID() + ", doc IDX= " + doc.get("IDX") + ", start position = " + spans.startPosition() + ", end " +
                                "position = " + spans.endPosition());
                    }
                }
            }
            //================================================================
            // 输出结果是
            // doc id = 0, doc IDX= 1, start position = 24, end position = 26
            //================================================================
            System.out.println();
            //修改slop，设置1，默认是0
            spanNearQuery = newOrderedNearQuery("title").addClause(first).addClause(second).setSlop(1).build();
            weight = spanNearQuery.createWeight(indexSearcher, true);
            leaves = indexSearcher.getIndexReader().getContext().leaves();
            for (LeafReaderContext leaf : leaves) {
                Spans spans = weight.getSpans(leaf, SpanWeight.Postings.POSITIONS);
                while (spans.nextDoc() != spans.NO_MORE_DOCS) {
                    Document doc = leaf.reader().document(spans.docID());
                    while (spans.nextStartPosition() != spans.NO_MORE_POSITIONS) {
                        System.out.println("doc id = " + spans.docID() + ", doc IDX= " + doc.get("IDX") + ", start position = " + spans.startPosition() + ", end " +
                                "position = " + spans.endPosition());
                    }
                }
            }
            //================================================================
            // 输出结果是
            // doc id = 0, doc IDX= 1, start position = 14, end position = 17
            // doc id = 0, doc IDX = 1, start position = 24, end position = 26
            //================================================================
        }
    }
}
```
#### 实现逻辑与查询
通过上面的图，想必你也知道，Lucene官方并不对SpanAndQuery提供支持，在Lucene的官方讨论组中，有人发起过[支持SpanAndQuery的issue](https://issues.apache.org/jira/browse/LUCENE-3371)，但是一直没有获得官方的回应。不过已经有商业公司实现了这种搜索技术，公司名称是[SearchTechnologies](https://www.searchtechnologies.com/)，API参见[SpanAndQuery](https://wiki.searchtechnologies.com/javadoc/qpl0.3snap/index.html?com/searchtechnologies/qpl/solr/queryoperators/SpanAndQuery.html)，但是并不开源（So Sad），我没有找到其实现的具体源码，如果你知道的话，烦请告知我一下。

既然官方不予支持，那就只能自己造轮子了，逻辑上来讲，也不复杂，有两种方式可以实现。

第一种方式，从词的角度，例如『爱你』和『你中』两个搜索词实现逻辑与，那么只要分别地把每一个搜索词都单独搜一下，最后在命中结果中取交集，就可以实现逻辑与的功能，代码写起来也很简单，在此不予示例。

第二种方式，从Term的角度，例如『爱你』如果按字切分，那么能够切成两个Term，分别是『爱』和『你』，这时候使用SpanNearQuery构造查询语句，加上一个很大的slop，但是不管slop多大，它总是有上限的，万一两个Term之间的距离超过slop，同样无法命中，所以说这种实现方式是存在漏洞的，除非你确定你的两个Term之间的距离不会超过某个具体的slop值，那么可以使用之。

注意在使用SpanNearQuery获取positions信息的时候，你不能够同时保证按字切分，又可以在两个搜索词之间设置slop值，这是因为如果用BooleanQuery去包装两个SpanNearQuery，那么将丢失positions信息。如果不按字切分，那么切出来的某个词就是一个Term，即将『爱你』和『你中』看成是两个Term，这时候是可以设置slop值的。如果按字切分，那么切成『爱|你』和『你|中』，实现逻辑与，如果设置slop值，相当于是『爱|slop值|你|slop值|你|slop值|中』，已经与逻辑与不匹配了，逻辑与的本意是『爱|slop值为0|你|slop为任意值|你|slop值为0|中』，请细细体会。

#### 实现逻辑或查询
官方已经对或逻辑提供了支持，就是SpanOrQuery，直接操练起来即可。
```java
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.FieldType;
import org.apache.lucene.document.LongField;
import org.apache.lucene.index.*;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.spans.*;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;
import java.util.List;

import static org.apache.lucene.search.spans.SpanNearQuery.newOrderedNearQuery;

/**
 * <p>
 * Created by wangxu on 2017/10/13 14:29.
 * </p>
 * <p>
 * Description: TODO
 * </p>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
 * WebSite: http://codepub.cn <br>
 * Licence: Apache v2 License
 */
public class SpanOrQueryDemo {
    @org.junit.Test
    public void test() throws IOException {
        String input = "现有的中文分词算法可分为三大类：基于字符串匹配的类基分词方法、基于理解的分词方法和基于统计的分词方法。";
        RAMDirectory ramDirectory = new RAMDirectory();
        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new ReservePunctuationAnalyzer());
        try (IndexWriter indexWriter = new IndexWriter(ramDirectory, indexWriterConfig)) {
            Document document = new Document();
            FieldType fieldType = new FieldType();
            fieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS);
            Field field = new Field("title", input, fieldType);
            LongField IDX = new LongField("IDX", 1, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法是很难很复杂滴";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 2, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法可以大幅度提升程序性能";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 3, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);
            indexWriter.commit();
            IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));

            SpanTermQuery first = new SpanTermQuery(new Term("title", "类"));
            SpanTermQuery second = new SpanTermQuery(new Term("title", "基"));
            SpanNearQuery spanNearQueryFirst = newOrderedNearQuery("title").addClause(first).addClause(second).build();

            first = new SpanTermQuery(new Term("title", "算"));
            second = new SpanTermQuery(new Term("title", "法"));
            SpanNearQuery spanNearQuerySecond = newOrderedNearQuery("title").addClause(first).addClause(second).build();
            SpanOrQuery spanOrQuery = new SpanOrQuery(spanNearQueryFirst, spanNearQuerySecond);

            SpanWeight weight = spanOrQuery.createWeight(indexSearcher, true);
            List<LeafReaderContext> leaves = indexSearcher.getIndexReader().getContext().leaves();
            for (LeafReaderContext leaf : leaves) {
                Spans spans = weight.getSpans(leaf, SpanWeight.Postings.POSITIONS);
                while (spans.nextDoc() != Spans.NO_MORE_DOCS) {
                    Document doc = leaf.reader().document(spans.docID());
                    while (spans.nextStartPosition() != Spans.NO_MORE_POSITIONS) {
                        System.out.println("doc id = " + spans.docID() + ", doc IDX= " + doc.get("IDX") + ", start position = " + spans.startPosition() + ", end " +
                                "position = " + spans.endPosition());
                    }
                }
            }
            //================================================================
            // 输出结果是
            // doc id = 0, doc IDX= 1, start position = 7, end position = 9
            // doc id = 0, doc IDX= 1, start position = 24, end position = 26
            // doc id = 1, doc IDX= 2, start position = 3, end position = 5
            // doc id = 2, doc IDX= 3, start position = 3, end position = 5
            //================================================================
        }
    }
}
```

### 实现SpanAllNearQuery
这是一个附加功能，因为目前还没有碰到这样的需求，但是这种查询实现起来非常有意思，所以在此简单讲解一下。这个问题的来源是有人提了个[issue](https://issues.apache.org/jira/browse/LUCENE-3371)，请求官方支持SpanAllNearQuery，但是同样地，官方不理不睬。果然公益的就是拽啊，完全不倾听用户的需求，不像商业公司，为了赚用户的钱，只要用户有需求，就尽力实现。

那么这个需求是什么样的呢？简单表示如下`a WITHIN 5 WORDS OF (b AND c)`，还可以把它换一种方式理解`(a WITHIN 5 WORDS OF b) AND (a WITHIN 5 WORDS OF c)`，就是说我要查询，在a的前面5个或者后面5个token中出现b和c的所有结果集。要实现这个功能，需要借助于SpanNotQuery和SpanOrQuery，SpanOrQuery在实现或逻辑中已经介绍过了，那么SpanNotQuery又是什么意思呢？举例如下
>SpanNotQuery(a, b, 5, 5)表示在a的前5个或者后5个token中不能出现b
SpanNotQuery(a, c, 5, 5)表示在a的前5个或者后5个token中不能出现c

下面先从逻辑上先实现这个需求，要获得在a的前面或后面5个token中出现b和c，需要将其反转理解，先查询在a的前面5个或者后面5个token中不能出现b『SpanNotQuery(a, b, 5, 5)』或者在a的前面5个或者后面5个token中不能出现c的结果『SpanNotQuery(a, c, 5, 5)』，再用SpanOrQuery来组合『SpanNotQuery(a, b, 5, 5)』和『SpanNotQuery(a, c, 5, 5)』实现或逻辑，最后用SpanNotQuery排除掉SpanOrQuery的结果集，那么剩下的就是在a的前面5个或者后面5个能出现b也能出现c的结果。
```java
import org.apache.lucene.analysis.core.WhitespaceAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.IntField;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.index.Term;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.search.spans.SpanNotQuery;
import org.apache.lucene.search.spans.SpanOrQuery;
import org.apache.lucene.search.spans.SpanTermQuery;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

/**
 * <p>
  * Created by wangxu on 2017/06/16 16:02.
 * </p>
  * <p>
  * Description: TODO
  * </p>
  *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0 <br/>
  * WebSite: http://codepub.cn <br>
  * Licence: Apache v2 License
 */public class SpanAllNearQueryDemo {
    @org.junit.Test
  public void test() throws IOException {
        RAMDirectory ramDirectory = new RAMDirectory();
  IndexWriter indexWriter = new IndexWriter(ramDirectory, new IndexWriterConfig(new WhitespaceAnalyzer()));
  Document document = new Document();
  document.add(new TextField("key", "X b X X X X a X X X X c X", Field.Store.YES));//命中
  document.add(new IntField("IDX", 1, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "X X X X X b a c X X X X X", Field.Store.YES));//命中
  document.add(new IntField("IDX", 2, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "X b X X X X a a X X X X c", Field.Store.YES));//不命中，不能同时以两个a为中心，两个a必选其一
  document.add(new IntField("IDX", 3, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "X b X X X X X a X X X X c", Field.Store.YES));//不命中
  document.add(new IntField("IDX", 4, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "X b X X X X a X X X X X c", Field.Store.YES));//不命中
  document.add(new IntField("IDX", 5, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "b X X X X X a X X X X X c", Field.Store.YES));//不命中
  document.add(new IntField("IDX", 6, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "b X X X X X a X X X X X X", Field.Store.YES));//不命中
  document.add(new IntField("IDX", 7, Field.Store.YES));
  indexWriter.addDocument(document);

  document = new Document();
  document.add(new TextField("key", "X X X X X X a X X X X X X", Field.Store.YES));//不命中
  document.add(new IntField("IDX", 8, Field.Store.YES));
  indexWriter.addDocument(document);
  indexWriter.commit();

  IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(indexWriter));
  SpanTermQuery a = new SpanTermQuery(new Term("key", "a"));
  SpanTermQuery b = new SpanTermQuery(new Term("key", "b"));
  SpanTermQuery c = new SpanTermQuery(new Term("key", "c"));
  SpanOrQuery exclude = new SpanOrQuery(new SpanNotQuery(a, b, 5, 5), new SpanNotQuery(a, c, 5, 5));
  //排除在a的前5个或者后5个不能出现b也不能出现c的document，那么剩下的就是在a的前5个token或者后5个token能够出现b和c的document
  SpanNotQuery spanNotQuery = new SpanNotQuery(a, exclude);
  TopDocs search = indexSearcher.search(spanNotQuery, Integer.MAX_VALUE);
  ScoreDoc[] scoreDocs = search.scoreDocs;
 for (ScoreDoc scoreDoc : scoreDocs) {
            System.out.println("hist IDX: " + indexSearcher.doc(scoreDoc.doc).get("IDX"));
  }
        indexSearcher.getIndexReader().close();
  indexWriter.close();
  }
}
```

### SpanNearQuery实现通配符查询
```java
import com.yuewen.nrzx.character.analyzer.ReservePunctuationAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.FieldType;
import org.apache.lucene.document.LongField;
import org.apache.lucene.index.*;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.search.WildcardQuery;
import org.apache.lucene.search.spans.SpanMultiTermQueryWrapper;
import org.apache.lucene.search.spans.SpanNearQuery;
import org.apache.lucene.search.spans.SpanQuery;
import org.apache.lucene.search.spans.SpanTermQuery;
import org.apache.lucene.store.RAMDirectory;

import java.io.IOException;

public class SpanNearQueryAndWildcardQueryDemo {
    @org.junit.Test
    public void test() throws IOException {
        String input = "现有的中文分词算法可分为三大类：基于字符串匹配的类基分词方法、基于理解的分词方法和基于统计的分词方法。";
        RAMDirectory ramDirectory = new RAMDirectory();
        IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new ReservePunctuationAnalyzer());
        try (IndexWriter indexWriter = new IndexWriter(ramDirectory, indexWriterConfig)) {
            Document document = new Document();
            FieldType fieldType = new FieldType();
            fieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS);
            Field field = new Field("title", input, fieldType);
            LongField IDX = new LongField("IDX", 1, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法是很难很复杂滴";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 2, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);

            input = "计算机算法可以大幅度提升程序性能";
            document = new Document();
            field = new Field("title", input, fieldType);
            IDX = new LongField("IDX", 3, Field.Store.YES);
            document.add(field);
            document.add(IDX);
            indexWriter.addDocument(document);
            indexWriter.commit();
            IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(ramDirectory));
            // 用?和*均可以实现SpanNearQuery的通配符查询，但是注意*在通配符查询中表示可以匹配0个或多个字符
            // 但是在SpanQuery中只能匹配相当于slop=1的情形，不能匹配slop大于1的情形
            SpanTermQuery first = new SpanTermQuery(new Term("title", "复"));
            SpanQuery wildcard = new SpanMultiTermQueryWrapper<>(new WildcardQuery(new Term("title", "?")));
            SpanTermQuery last = new SpanTermQuery(new Term("title", "滴"));

            SpanNearQuery spanNearQuery = new SpanNearQuery.Builder("title", true).addClause(first).addClause(wildcard).addClause(last).build();
            TopDocs search = indexSearcher.search(spanNearQuery, Integer.MAX_VALUE);
            System.out.println("IDX: " + indexSearcher.doc(search.scoreDocs[0].doc).get("IDX"));

            wildcard = new SpanMultiTermQueryWrapper<>(new WildcardQuery(new Term("title", "*")));
            spanNearQuery = new SpanNearQuery.Builder("title", true).addClause(first).addClause(wildcard).addClause(last).build();
            search = indexSearcher.search(spanNearQuery, Integer.MAX_VALUE);
            System.out.println("IDX: " + indexSearcher.doc(search.scoreDocs[0].doc).get("IDX"));
        }
    }
}
```

### 实验
服务器 CPU 以及内存信息
> $ cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c

`24  Intel(R) Xeon(R) CPU E5-2420 v2 @ 2.20GHz`

> $ free -g

||total|used|free|shared|buffers|cached|
|--|--|--|--|--|--|
|Mem|62|56|6|0|0|1|
|-/+ buffers/cache|54|8|||||
|Swap|1|0|1|||||

在公司内部，仅仅索引了十分之一的文档（Document数量：20023911），鉴于没有存储Field，也没有存储TermVectors，索引不算太大，简单测试了下，如果存储TermVectors的话，索引会从112GB增长到162GB，如果再存储Field的话，那么索引要超过200GB。此处的实验只是简单的单次搜索，没有测试与逻辑和或逻辑情况下的搜索情况。搜索阶段实验的结果如下所示

| 线程数目  |搜索总次数|命中次数|搜索总耗时|平均单次耗时|搜索加构建Query耗时|平均单次耗时|索引大小|
| ------- | ------ | ------ | ------ | ----- | ----- | ----- | ----- |
| 1  | 18820  |18139|36001,698ms|1912.95ms|36011,989ms|1913.50ms|112GB|
| 5  | 18820  |18139|30775,283ms|1635.24ms|30785,538ms|1635.79ms|112GB|
| 10  | 18820  |18139|21637,515ms|1149.71ms|21647,953ms|1150.26ms|112GB|
| 50  | 18820  |18139|21572,506ms|1146.25ms|21583,101ms|1146.82ms|112GB|

索引阶段并没有做详细完备的实验，只是简单拉取了一点数据，记录如下，仅供参考。索引2080550个Document，统计从数据库拉取数据加更新数据到索引耗时2133s，平均每次耗时0.001s。只计算更新数据到索引中，不计算从数据库拉取数据耗时464s，平均每次耗时0.0002s，可见索引速度也是很快的，这全部得益于Lucene的优良设计。
