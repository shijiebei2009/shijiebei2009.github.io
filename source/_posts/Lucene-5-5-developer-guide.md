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
- **BooleanQuery**对其中包含的查询子句（调用一次**add()**方法可以看作一个子句）是有数量限制的，默认情况下允许包含1024个查询子句，该限制主要是基于性能方面的考虑。当子句数量超过最大值时，程序会抛出**TooManyClauses**异常。如果你在一些特殊情况下，需要增大查询子句的数量限制，可以调用`setMaxClauseCount(int maxClauseCount)`方法进行设置，但是该操作明显会对性能产生影响
- 举例说明**PhraseQuery**匹配中**slop**的计算方式，这里的**slop**是指若要按顺序组成给定的短语所需要移动位置的次数。如**Amsterdam has lots of bridges**这句话，假如我需要查询**has**和**bridges**，那么它们之间的**slop**为2，假如我需要查询**bridges**和**has**（注意两次查询添加短语的顺序不同），那么它们之间的**slop**为4，这个4代表的是将**has**移动到**bridges**之后需要的步数
- **PhraseQuery**支持复合项短语。无论短语中有多少个项，**slop**因子都规定了按顺序移动项位置的总次数的最大值。例如
```java
//待检索文本：Venice has lots of canals
PhraseQuery build = new PhraseQuery.Builder().setSlop(1).add(new Term("contents", "Venice")).add(new Term("contents", "lots")).add(new Term("contents", "canals")).build();
search = indexSearcher.search(build, 10);
Assert.assertNotEquals(1, search.totalHits);//移动总次数为1+1=2，所以NotEquals
```
- 在使用通配符**WildcardQuery**进行查询时，可能会降低系统的性能，较长的前缀可以减少用于查找匹配搜索枚举的项的个数，如以通配符为首的查询模式会强制枚举所有索引中的项以用于搜索匹配
- 类似查询**FuzzyQuery**会尽可能地枚举出一个索引中所有项，因此，最好尽量少地使用这类查询，即便要使用这类查询，起码也应当知晓其工作原理以及它对程序性能的影响
- **QueryParser**中针对某个项的否定操作必须与至少一个非否定项的操作联合起来进行，否则程序不会返回结果文档，换句话说不能使用**“NOT term”**而必须使用**“a AND NOT b”**
- 查询语句中用双引号括起来的项可以用来创建一个**PhraseQuery**，例如查询语句"This is some phrase\*"被**StandardAnalyzer**分析时，将被解析成用短语"some phrase"构成的**PhraseQuery**对象。星号不能被解释成模糊查询，请记住：双引号内的文本会促使分析器将之转换为PhraseQuery。那么说了这么多废话是什么意思呢？举例说明
```java
Query query = new QueryParser("field", new StandardAnalyzer()).parse("\"This is some phrase*\"");
Assert.assertEquals("analyzed", "\"? ? some phrase\"", query.toString("field"));
System.out.println(query.toString("field"));
//输出"? ? some phrase"，停用词This is被?代替了，而引号中的*被忽略了
```
- 分析器介绍
 - **WhitespaceAnalyzer**：顾名思义，就是通过空格来分割文本信息，而并不对生成的语汇单元进行其他的规范化处理
 - **SimpleAnalyzer**：该分析器会首先通过非字母字符来分割文本信息，然后将语汇单元统一为小写形式，需要注意的是，该分析器会去掉数字类型的字符，但会保留其他字符
 - **StopAnalyzer**：该分析器功能与**SimpleAnalyzer**类似，区别在于，前者会去除停用词，例如英文中的**the**、**a**等
 - **StandardAnalyzer**：这是Lucene最复杂的核心分析器。包含大量的逻辑操作来识别某些种类的语汇单元，比如公司名称、E-Mail地址以及主机名称等。它还会将语汇单元转换成小写形式，并去除停用词和标点符号，经试验，Lucene5.5中，该分析器无法切分出完整的邮箱地址
 - **KeywordAnalyzer**：将整个文本作为一个单一语汇单元处理
- **MultiPhraseQuery**类与**PhraseQuery**类似，区别在于前者允许在同一位置上针对多个项的查询，当然要实现这种查询也可以使用**BooleanQuery**类或者逻辑”或“连接符将所有可能的短语进行联合查询，但是这样会以更高昂的系统消耗为代价
- 在进行多域值分析时，文档可能包含同名的多个**Field**实例，而**Lucene**在索引过程中在逻辑上将这些域的语汇单元按照顺序附加在一起，举例来说，如果一个域值是"***It's time to pay incom tax***"而下一个域值是"***return library books on time***"，那么对于短语"***tax return***"的搜索将会匹配到这个域。为了解决这个问题，必须通过继承**Analyzer**类来创建自己的分析器，然后重载**getPositionIncrementGap**方法（以及**tokenStream**或**resuableTokenStream**方法）。默认情况下，**getPositionIncrementGap**方法会返回0（无间隙），意思是它在运行时会认为各个域值是连接在一起的，如果将这个域值增加到足够大，那么位置查询就不会错误地在各个域值边界进行匹配了。注意一点，如果程序要高亮显示这些域，那么错误的偏移量会导致程序将错误的文本部分进行高亮显示，语汇单元**OffsetAttribute**对象提供了几个方法来检索偏移量的起始和结束，还提供了一个特殊的**endOffset**方法，该方法主要用于返回域的最后偏移
- 另一种针对多域值的查询是使用**MultiFieldQueryParser**，它是**QueryParser**的子类，它会在后台程序中实例化一个**BooleanQuery**对象，并将每个域的查询结合起来。当程序向**BooleanQuery**添加查询子句时，默认操作符**OR**被用于最简单的解析方法中，也可以通过使用**parse**方法指定**BooleanClause.Occur**的值。**MultiFieldQueryParser**有一些使用限制，这主要是由它使用**QueryParser**的方法导致的，你只能使用**QueryParser**的一些默认设置，而不能更改之。**MultiFieldQueryParser**的一个重要缺陷就是它会生成更多的复杂查询，而**Lucene**必须对每条查询分别进行测试，这样会比使用全包含域速度更慢
- 多域值查询的第三种方式是使用**DisjunctionMaxQuery**，它会封装一个或多个任意的查询，将匹配的文档进行**OR**操作，一个有趣的地方是当某个文档匹配到多于一条查询时，该类会将这个文档的评分记为最高分，而与**BooleanQuery**相比，后者会将所有匹配的评分相加。**DisjunctionMaxQuery**还包含一个可选的仲裁器，因此所有处理都是平等的，一个匹配更多查询的文档能够获得更高的评分
- 如果希望停止较慢的搜索，那么可以使用**TimeLimitingCollector**，它能在耗时较长的情况下停止搜索，显然你必须针对超时搜索情况选择具体的处理内容
- **TimeLimitingCollector**在收集搜索结果时添加一些自己的操作（比如每当获取一个搜索结果文档时都检查此时是否已超时），这会使得搜索速度变得稍慢，其次，它只在收集搜索结果时判断是否超时，然后一些查询有可能在**Query.rewrite()**操作期间消耗较长时间。对于这类查询来说，程序可能只有在搜索超时时才能捕获**TimeExceededException**异常
- **Lucene**通过创建抽象基类**FieldComparatorSource**的子类来实现自定义排序
- **Lucene**所有的核心搜索方法都在后台使用**Collector**子类来完成搜索结果收集。例如，当通过相关性进行排序时，后台会使用**TopScoreDocCollector**类，当通过域进行排序时，则使用**TopFieldCollector**
- Lucene索引前缀后缀规则：所谓前缀后缀规则，即当某个词和前一个词有共同的前缀的时候，后面的词仅仅保存前缀在词中的偏移（offset），以及除前缀以外的字符串（称为后缀）。例如：**Term Terminal**存储为`Term4inal`
- 索引差值规则：先后保存两个整数的时候，后面的整数仅仅保存和前面整数的差值即可，例如：**5,9,11**存储为`|5|4(9-5)|3(11-9)|`
- 关于Directory子类的选择问题，最简单的方式是使用`public static FSDirectory open(Path path)`，让Lucene来替你尽力选择最适合当前硬件环境的实现

#### Lucene事务相关
- 通过IndexWriter.getReader()获得的Reader是能看到上次commit之后，IndexWriter执行至当前的所有变化，该方法提供一种"near real-time"近实时的搜索
- 当IndexWriter正在做更改的时候，所有更改都不会对当前搜索该索引的IndexReader可见，直到你commit或者打开了一个新的NRT(near real-time)Reader。一次只能有一个IndexWriter实例对索引进行更改
- 仅在一个新的目录上打开一个IndexWriter并不会产生一个空的commit操作，即你无法在这个目录上打开一个IndexReader，直到你完成一次commit
- 单个Lucene索引可以保存多个commit，每次commit都持有该commit被创建的时间点的索引视图
- KeepOnlyLastCommitDeletionPolicy是默认的删除机制实现，它会删除掉除了最近一次commit以外的所有commit。如果采用NoDeletionPolicy，那么每一次commit都会保存
- 你可以在commit的时候传送userData`(Map<String,String>)`，用于记录关于本次commit的一些用户自定义信息（对Lucene不透明），然后使用IndexReader.listCommits方法获得索引的所有commit信息。一旦你找到了一次commit，你可以在其上找开一个IndexReader对commit点上的索引数据进行搜索
- 你也可以基于之前的commit打开一个IndexWriter，回滚之后的所有变动。这有点类似于rollback方法，不同之处在于它允许你在多个commit间rollback，而不仅是回滚当前IndexWriter会话中所做的变动

#### Lucene的QueryParse表达式说明
|表达式   | 匹配文档  |
| ------------ | ------------ |
|java|在字段中包含java|
|java junit<br/>java OR junit|在字段中包含java或者junit|
|+java +junit<br/>java AND junit|在字段中包含java以及junit|
|title:ant|在title字段中包含ant|
|title:extreme<br/>-subject:sports<br/>title:extreme<br/>AND NOT subject:sports|在title字段中包含extreme并且在subject字段中不能包含sports|
|(agile OR extreme) AND methodology|在字段中包含methodology并且同时包括agile或者extreme|
|title:"junit in action"|在title字段中包含junit in action|
|title:"junit action"~5|junit和action之间距离小于5的文档|
|java\*|包含以java开头的，例如：javaspaces,javaserver|
|java~|包含和java相似的，如lava|
|lastmodified:[1/1/04 TO 12/31/04]|在lastmodified字段中值为2004-01-01到2004-12-31中间的|
|+pubdate:[10100101 TO 20101231] Java AND (Lucene OR Apache)|检索2010年出版的关于Java、且内容中包含Lucene或Apache关键字的所有书籍|

**注意：**如果查询表达式使用了以下特殊字符，你就应该对其进行转义操作，使这些字符在一般的表达式中能够发挥作用。**QueryParser**在各个项中使用反斜杠`(\)`来表示转义字符。需要进行转义的字符有：`\ + - ! ( ) : ^ ] { } ~ * ?`

#### Lucene常见Field
|表达式   | 匹配文档  |
| ------------ | ------------ |
|IntField|主要对int类型的字段进行存储，需要注意的是如果需要对IntField进行排序使用SortField.Type.INT来比较，如果进范围查询或过滤，需要采用NumericRangeQuery.newIntRange()|
|LongField|主要处理Long类型的字段的存储，排序使用SortField.Type.Long,如果进行范围查询或过滤利用NumericRangeQuery.newLongRange()，LongField常用来进行时间戳的排序，保存System.currentTimeMillions()|
|FloatField|对Float类型的字段进行存储，排序采用SortField.Type.Float,范围查询采用NumericRangeQuery.newFloatRange()|
|BinaryDocVluesField|只存储不共享值，如果需要共享值可以用SortedDocValuesField|
|NumericDocValuesField|用于数值类型的Field的排序(预排序)，需要在要排序的field后添加一个同名的NumericDocValuesField|
|SortedDocValuesField|用于String类型的Field的排序，需要在StringField后添加同名的SortedDocValuesField|
|StringField|用于String类型的字段的存储，StringField是只索引不分词|
|TextField|对String类型的字段进行存储，TextField和StringField的不同是TextField既索引又分词|
|StoredField|存储Field的值，可以用IndexSearcher.doc和IndexReader.document来获取此Field和存储的值|

参考资料
【1】Lucene In Action (Second Edition)
【2】http://3dobe.com/archives/172/
【3】http://blog.mikemccandless.com/2012/03/transactional-lucene.html