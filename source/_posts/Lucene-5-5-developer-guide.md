title: Lucene 5.5 开发手册
date: 2016-08-12 22:52:23
tags: [Lucene]
categories: Programming Notes

---

#### Lucene和Solr的历史版本
[Lucene历史版本](http://archive.apache.org/dist/lucene/java/)，不妨点进去看看，会发现**Lucene**的版本更新很频繁，所以**Lucene**的**Doc**注释比**JDK**的**Doc**注释差太多，在研读`Lucene In Action`的过程中，发现此书的**Lucene**版本为**3.0**，而自己使用的**Lucene**版本是**5.5**，所以会有诸多冲突之处，现聊记之，以备查用。另外附上[Solr历史版本](http://archive.apache.org/dist/lucene/solr/)。

在学习**Lucene**过程中，官方推荐的**Lucene**索引查看工具是`Luke`，下载地址[点我](https://github.com/DmitryKey/luke/releases)。

#### Lucene API变动相关
- **Field**类中的枚举**Index**已被废弃，转而采用**FieldType**，并通过**setIndexOptions**方法设置索引选项
- **IndexWriter**的**optimize**方法已被删除，推荐用**forceMerge**方法代替
- 对**Document**加权在**4.x**版本之后已经删除，如果想要对文档加权，需要对该文档中的每个**Field**都进行加权
- **NumericField**被删除，由**FieldType**类中的枚举**NumericType**进行设置，**NumericType**提供了四种类型的数值，分别是**INT**，**LONG**，**FLOAT**，**DOUBLE**
- **IndexWriter.MaxFieldLength**等类似的一系列常量全被删除
- **IndexWriter**中的**isLocked**方法和**unlock**方法全被删除，使用**Directory**类中的**obtainLock**方法替代，调用方式如下`Directory.obtainLock(IndexWriter.WRITE_LOCK_NAME)`，如果已经获得锁的话，再次获取会抛异常`LockObtainFailedException:lock instance already obtained`
- **IndexWriter**中的**setInfoStream(System.out)**方法被转移到**IndexWriterConfig**类中，该方法用来打印**Lucene**操作索引的相关输出信息
- **BooleanQuery**的构造函数被废弃，使用**BooleanQuery.Builder**替代之
- **BooleanQuery**中的**add()**被废弃，推荐使用**BooleanQuery.Builder().add()**代替之，示例如下
```java
TermQuery termQuery = new TermQuery(new Term("country", "Italy"));
TermQuery termQuery1 = new TermQuery(new Term("city", "Venice"));
BooleanQuery build = new BooleanQuery.Builder().add(termQuery,
BooleanClause.Occur.MUST).add(termQuery1, BooleanClause.Occur.MUST).build();
```
  其中**Occur**有三种选项，分别是**MUST**表只有匹配该查询子句的文档才在考虑之列，**SHOULD**意味着该项只是可选项，**MUST_NOT**意味着搜索结果不会包含任何匹配该查询子句的文档。使用**SHOULD**可以实现逻辑或（**OR**）查询。
- **PhraseQuery**中的构造函数和**setSlop**方法被废弃，推荐使用`PhraseQuery build = new PhraseQuery.Builder().setSlop(2).build();`代替
- 同样**PhraseQuery**中**add()**方法废弃，推荐使用`new PhraseQuery.Builder().setSlop(2).add(term).add(term1).build()`代替，同时**slop**中的数值代表的是**term**和**term1**中间的字符个数，不包含**term**和**term1**在内。注意**slop**的默认值时0
- **MatchAllDocsQuery(String normsField)**构造函数被删除，仅保留一个无参构造
- 内建的**TermAttribute**被删除，推荐使用**CharTermAttribute**代替之，**TermAttributeImpl**被删除，推荐使用**CharTermAttributeImpl**代替之
- **PerFieldAnalyzerWrapper.addAnalyzer()**方法被删除，使用构造函数初始化指定域所对应的分析器
```java
Map<String, Analyzer> fields = new HashMap<>();
fields.put("partnum", new KeywordAnalyzer());
//对于其他的域，默认使用SimpleAnalyzer分析器，对于指定的域partnum使用KeywordAnalyzer
PerFieldAnalyzerWrapper perFieldAnalyzerWrapper = new PerFieldAnalyzerWrapper(new SimpleAnalyzer(), fields);
```
- 域缓存加载所有文档中某个特定域的值到内存，便于随机存取该域值。域缓存只能用于包含单个项的域，也就是说使用域缓存的文档都必须拥有一个与指定域对应的单一域值，即域值不能被拆分。**FieldCache**在**Lucene 5.5.0**中被**lucene-core**模块移除，转移到了**lucene-misc**模块并且访问权限为包级私有，**FieldCache**类已经不鼓励使用，当要在某个**Field**上进行排序时，用户应该使用**Doc**值来索引相应的**Field**，相对于使用**FieldCache**更加快且耗费较少的堆大小
- **IndexSearcher**中**setDefaultFieldSortScoring()**方法被移除
- **SpanQuery.getSpans()**方法转移到**SpanWeight**中，在调用该方法前，需要先调用**createWeight**方法创建**SpanWeight**实例
- **SpanWeight**中的**getSpans()**方法调用示例如下
```java
Spans spans = weight.getSpans(indexReader.getContext().leaves().get(0), SpanWeight.Postings.POSITIONS);
```
- 如果想在域的起点查询指定的跨度，需要使用**SpanFirstQuery(SpanQuery match, int end)**，**end**指定了跨度的结束位置
- 搜索过滤中的一系列过滤器都被废弃或删除，例如**TermRangeFilter**、**NumericRangeFilter**、**QueryWrapperFilter**、**PrefixFilter**、**CachingWrapperFilter**等，推荐使用相应的**\*Query**类和**BooleanClause.Occur.FILTER**替代之，**SpanQueryFilter**、**CachingSpanFilter**、**FieldCacheRangeFilter**等被移除
- **FilteredDocIdSet**和**FilteredQuery**将在**Lucene 6.0**中移除
- **MultiSearcher**被删除，用**MultiReader**代替
- **getTermFreqVector**被删除，用**getTermVector**代替
- **FieldSelector**被删除，用**StoredFieldVisitor**代替，**FieldSelector**的子类**LoadFirstFieldSelector**、**MapFieldSelector**、**SetBasedFieldSelector**亦被删除
- **TermVectorMapper**被删除，相应地其子类**PositionBasedTermVectorMapper**、**SortedTermVectorMapper**、**FieldSortedTermVectorMapper**亦被删除
- **FieldSelectorResult**被删除，**TermVector**被废弃

#### Lucene 开发相关
- 索引数字时，需要选择一个不丢弃数字的分析器，**WhitespaceAnalyzer**和**StandardAnalyzer**可以作为候选，而**SimpleAnalyzer**和**StopAnalyzer**两个类会将语汇单元流中的数字剔除
- 优化索引只能提高搜索速度，而不是索引速度。注意索引优化会消耗大量的**CPU**和**I/O**资源，在优化期间，索引会占用较大的磁盘空间，大约为优化初期的3倍
- 对于一个索引来说，一次只能打开一个**Writer**，**Lucene**采用文件锁来提供保障，该锁只有当**IndexWriter**对象被关闭时才会释放
- 任意多个线程都可以共享同一个**IndexReader**类或者**IndexWriter**类，这些类不仅是线程安全的，而且是线程友好的，能够很好的扩展到新增线程，针对并发访问唯一的限制是不能同时打开多于一个**Writer**
- 如果选择自己实现锁机制，要确认该机制能够正确运行，可以使用简单的调试工具，**LockStressTest**类，该类可以与**LockVerifyServer**类和**VerifyingLockFactory**类联合使用，以确认你自己实现的锁机制能正常运行
- 刷新操作是用来释放被缓存的更改的，而提交操作是用来让所有的更改（包括被缓存的更改或者已经刷新的更改）在索引中保持可见
- 打开**IndexReader**需要较大的系统开销，因此必须尽可能重复使用一个**IndexReader**实例以用于搜索，并限制打开新**IndexReader**的频率
- **IndexSearcher**实例只能对其初始化时刻所对应的索引进行搜索。如果索引操作是由多线程完成的，新编入索引的文档则不能被该**searcher**看到
- Query的一些子类说明：
 - 通过项进行搜索：**TermQuery**
 - 在指定的项范围内搜索：**TermRangeQuery**
 - 通过短语搜索：**PhraseQuery**
 - 通配符查询：**WildcardQuery**
 - 搜索类似项：**FuzzyQuery**
 - 匹配所有文档：**MatchAllDocsQuery**
- **BooleanQuery**对其中包含的查询子句（调用一次**add()**方法可以看作一个子句）是有数量限制的，默认情况下允许包含1024个查询子句，