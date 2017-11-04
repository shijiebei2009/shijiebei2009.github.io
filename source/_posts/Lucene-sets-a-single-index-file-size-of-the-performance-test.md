---
title: Lucene设置单索引文件大小之性能测试
date: 2016-10-09 21:31:36
tags: [Lucene]
categories: Programming Notes

---

### 环境

**Operating System**：Windows 7 企业版 64-bit (6.1, Build 7601) Service Pack 1
**Processor**：Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz (8 CPUs), ~3.6GHz
**Java version**："1.8.0_91" Java HotSpot(TM) 64-Bit Server VM (build 25.91-b15, mixed mode)
**Lucene Version**：V 5.5.0

### 合并策略
Lucene使用**TieredMergePolicy**作为其默认合并策略，这个默认的策略是合并那些差不多大小的段，受限于每次合并时允许的段的数量。该合并策略的一些默认属性如下

```java
public static final double DEFAULT_NO_CFS_RATIO = 0.1;
private int maxMergeAtOnce = 10;//在一次合并过程中最多合并十个segments
private long maxMergedSegmentBytes = 5*1024*1024*1024L;//合并之后的最大段文件为5GB
private int maxMergeAtOnceExplicit = 30;
private long floorSegmentBytes = 2*1024*1024L;
private double segsPerTier = 10.0;
private double forceMergeDeletesPctAllowed = 10.0;
private double reclaimDeletesWeight = 2.0;
```

这种默认的合并策略会导致单个段文件的大小达到5GB，那么如果想要控制单个段文件大小的上限要怎么办呢？追踪该类至其父类MergePolicy，继续追踪MergePolicy的所有子类，发现MergePolicy的子类包括：NoMergePolicy、MergePolicyWrapper、LogMergePolicy、TieredMergePolicy，查看LogMergePolicy发现该类有两个子类，分别为LogDocMergePolicy和LogByteSizeMergePolicy，顾名思义，前者考虑段文件的数量，后者考虑段文件的大小。

另外**TieredMergePolicy**和**LogByteSizeMergepolicy**区别在于前者可以合并不相邻的段，并且区分最多允许一次合并的段数`setMaxMergeAtOnce(int v)`和一层最多容许的段数`setSegmentsPerTier(double v)`。

至此，找到了我们需要使用的合并策略，即**LogByteSizeMergePolicy**。

### 复合索引文件
复合索引文件是指，除了段信息文件，锁文件，以及删除的文件外，其他的一系列索引文件压缩一个后缀名为cfs的文件，意思就是所有的索引文件会被存储成一个单例的Directory，而非复合索引是灵活的，可以单独的访问某几个索引文件，而复合索引文件则不可以，因为其压缩成了一个文件，所以在某些场景下能够获取更高的效率，比如说，查询频繁，而不经常更新的需求，就很适合这种复合索引格式。

如果仅仅设置`setUseCompoundFile(false)`，那么在生成的索引中，依然会有复合索引文件并且其大小会超过`setMaxMergeMB(double mb)`所设置的大小，是不是很奇怪，这是为什么呢？收先查看`setMaxMergeMB(double mb)`Doc说明如下：
>org.apache.lucene.index.LogByteSizeMergePolicy
public void setMaxMergeMB(double mb)
Determines the largest segment (measured by total byte size of the segment's files, in MB) that may be merged with other segments. Small values (e.g., less than 50 MB) are best for interactive indexing, as this limits the length of pauses while indexing to a few seconds. Larger values are best for batched indexing and speedier searches.
Note that setMaxMergeDocs is also used to check whether a segment is too large for merging (it's either or).

该函数用于控制可用于合并的最大的段文件大小，通常合并之后的段文件的大小均会超过此值。所以单靠此函数不能实现控制单索引文件上限的目标。

继续查看`setUseCompoundFile(false)`Doc说明如下：
>org.apache.lucene.index.LiveIndexWriterConfig
public LiveIndexWriterConfig setUseCompoundFile(boolean useCompoundFile)
Sets if the IndexWriter should pack newly written segments in a compound file. Default is true.
Use false for batch indexing with very large ram buffer settings.
**Note**: To control compound file usage during segment merges see **MergePolicy.setNoCFSRatio(double)** and **MergePolicy.setMaxCFSSegmentSizeMB(double)**. This setting only applies to newly created segments.

其中提到了`MergePolicy.setNoCFSRatio(double)`，继续查看该方法的Doc说明如下：
>org.apache.lucene.index.MergePolicy
public void setNoCFSRatio(double noCFSRatio)
If a merged segment will be more than this percentage of the total size of the index, leave the segment as non-compound file even if compound file is enabled. **Set to 1.0 to always use CFS regardless of merge size**.

继续追踪noCFSRatio查找到了预定义的默认值**DEFAULT_NO_CFS_RATIO**，该默认值为1.0，而根据Doc说明，当该值为1.0的时候，总是使用复合索引文件并且忽略合并大小的设置，所以就出现了设置不使用复合索引文件但无效的情况。
```java
protected static final double DEFAULT_NO_CFS_RATIO = 1.0;
protected double noCFSRatio = DEFAULT_NO_CFS_RATIO;
```
那么可以通过调用`logByteSizeMergePolicy.setNoCFSRatio();`去更改noCFSRatio的值，但是究竟设置多少合适，不得而知。此时不妨去关注下`setMaxCFSSegmentSizeMB(double)`方法，既然总是会有复合索引文件，那么通过设置复合索引文件的上限一样可以实现控制单索引文件大小的目的。该方法Doc说明如下：
>org.apache.lucene.index.MergePolicy
public void setMaxCFSSegmentSizeMB(double v)
If a merged segment will be more than this value, leave the segment as non-compound file even if compound file is enabled. Set this to Double.POSITIVE_INFINITY (default) and noCFSRatio to 1.0 to always use CFS regardless of merge size.

可知如果合并之后段文件大小会超过此值时，就保留该段文件为非复合索引文件，亦即不进行合并，所以此方法可以达到控制单索引文件上限的目标。

### 测试
使用不同的合并策略配合三种设置，添加50000000个Document，其中每个Document包含10个Field，这些Field类型包括Int、Long、String，其值随机生成。随后根据主键搜索1000000次，统计其累计搜索耗时，其中主键为String类型数字，主键值为[0，50000000)。

| 合并策略 |参数设置|索引文件数|索引总量|最大索引|搜索总耗时|单次搜索最大耗时|
| :---- | :---- | ----: | ----: | ----: | ----: | ----: |
| TieredMergePolicy  | 默认 |123| 7.33GB  |464MB|58.2s|140ms|
| LogByteSizeMergePolicy | LogByteSizeMergePolicy lbs;<br/>lbs.setMinMergeMB(1);<br/>lbs.setMaxMergeMB(64);|239|7.77GB|127MB|127.1s|314ms|
| LogByteSizeMergePolicy | IndexWriterConfig iwc;<br/>LogByteSizeMergePolicy lbs;<br/>iwc.setUseCompoundFile(false);<br/>lbs.setMinMergeMB(1);<br/>lbs.setMaxMergeMB(64);<br/> lbs.setMaxCFSSegmentSizeMB(64);|451|7.77GB|60.8MB|152.1s|446ms|

### 总结
虽然每次运行结果时间稍有不同，但总体趋势应该是不变的。即单索引文件上限越小，则生成的索引文件数量越多，索引文件数量越多，对应的单次搜索最长时间和总搜索时间均越长。所以应根据业务需求选择适当的合并策略，在满足需求之后尽可能提高搜索性能。