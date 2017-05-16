title: Lucene 6.0 索引文件格式
date: 2016-12-05 22:01:55
tags: [Lucene]
categories: Programming Notes

---

### 定义
在Lucene中基本的概念包括：index、document、field和term。一个index包含一个documents的序列
- 一个document是一个fields的序列
- 一个field是一个命名的terms序列
- 一个term是一个bytes的序列

在两个不同fields中的相同bytes序列被认为是不同的term。因此，term表示为一对：命名field的字符串，以及field内的bytes。

#### 倒排索引
谈到倒排索引，那么首先看看正排是什么样子的呢？假设文档1包含【中文、英文、日文】，文档2包含【英文、日文、韩文】，文档3包含【韩文，中文】那么根据文档去查找内容的话
- 文档1->【中文、英文、日文】
- 文档2->【英文、日文、韩文】
- 文档3->【韩文，中文】

反过来，根据内容去查找文档
- 中文->【文档1、文档3】
- 英文->【文档1、文档2】
- 日文->【文档1、文档2】
- 韩文->【文档2、文档3】

就是倒排索引，而Lucene擅长的也正在于此。

#### Fields的类型
- TextField：索引并分词，不包含词向量，多用于文本的正文
- StringField：索引但不分词，整个String作为单个标记索引，例如可用于“国家名称”或“ID”等，或者其它任何你想要用来排序的字段
- IntPoint：int型用于精确/范围查询的索引
- LongPoint：long型用于精确/范围查询的索引
- FloatPoint：float型用于精确/范围查询的索引
- DoublePoint：double型用于精确/范围查询的索引
- SortedDocValuesField：存储每个文档BytesRef值的字段，索引用于排序，如果需要存储值，需要再用StoredField实例
- SortedSetDocValuesField：存储每个文档一组BytesRef值的字段，索引用于faceting/grouping/joining，如果需要存储值，需要再用StoredField实例
- NumericDocValuesField：存储每个文档long值的字段，用于scoring/sorting/值检索
- SortedNumericDocValuesField：存储每个文档一组long值的字段，用于scoring/sorting/值检索
- StoredField：用于在汇总结果中检索的仅存储值

#### 段（Segments）
Lucene的索引可能是由多个子索引或Segments组成。每个Segment是一个完全独立地索引，可以单独用于搜索。索引涉及
1. 为新添加的documents创建新的segments
2. 合并已经存在的segments

搜索可能涉及多个segments或/和多个索引，每个索引可能由一组segments组成。

#### 文档编号
Lucene通过一个整型的文档编号指向每个文档，第一个被加入索引的文档编号为0，后续加入的文档编号依次递增。注意文档编号是可能发生变化的，所以在Lucene外部存储这些值时需要格外小心。

### 索引结构概述
每个segment索引包括信息
- Segment info：包含有关segment的元数据，例如文档编号，使用的文件
- Field names：包含索引中使用的字段名称集合
- Stored Field values：对于每个document，它包含属性-值对的列表，其中属性是字段名称。这些用于存储有关文档的辅助信息，例如其标题、url或访问数据库的标识符
- Term dictionary：包含所有文档的所有索引字段中使用的所有terms的字典。字典还包括包含term的文档编号，以及指向term的频率和接近度的指针
- Term Frequency data：对于字典中的每个term，包含该term的所有文档的数量以及该term在该文档中的频率，除非省略频率（IndexOptions.DOCS）
- Term Proximity data：对于字典中的每个term，term在每个文档中出现的位置。注意，如果所有文档中的所有字段都省略位置数据，则不会存在
- Normalization factors：对于每个文档中的每个字段，存储一个值，该值将乘以该字段上的匹配的分数
- Term Vectors：对于每个文档中的每个字段，可以存储term vector，term vector由term文本和term频率组成
- Per-document values：与存储的值类似，这些也以文档编号作为key，但通常旨在被加载到主存储器中以用于快速访问。存储的值通常用于汇总来自搜索的结果，而每个文档值对于诸如评分因子是有用的
- Live documents：一个可选文件，指示哪些文档是活动的
- Point values：可选的文件对，记录索引字段尺寸，以实现快速数字范围过滤和大数值（例如BigInteger、BigDecimal（1D）、地理形状交集（2D，3D））

### 文件命名
属于一个段的所有文件具有相同的名称和不同的扩展名。当使用复合索引文件，这些文件（除了段信息文件、锁文件和已删除的文档文件）将压缩成单个`.cfs`文件。当任何索引文件被保存到目录时，它被赋予一个从未被使用过的文件名字。


### 文件扩展名摘要
| 名称 | 文件扩展名 | 简短描述 |
| ------------ | ------------ |
| Segments File | segments_N | 保存了一个提交点（a commit point）的信息 |
| Lock File | write.lock | 防止多个IndexWriter同时写到一份索引文件中 |
| Segment Info| .si | 保存了索引段的元数据信息 |
| Compound File | .cfs，.cfe | 一个可选的虚拟文件，把所有索引信息都存储到复合索引文件中 |
| Fields | .fnm | 保存fields的相关信息 |
| Field Index | .fdx | 保存指向field data的指针|
| Field Data | .fdt | 文档存储的字段的值 |
| Term Dictionary | .tim |term词典，存储term信息 |
| Term Index | .tip | 到Term Dictionary的索引|
| Frequencies | .doc | 由包含每个term以及频率的docs列表组成 |
| Positions | .pos | 存储出现在索引中的term的位置信息 |
| Payloads | .pay | 存储额外的per-position元数据信息，例如字符偏移和用户payloads |
| Norms | .nvd，.nvm | .nvm文件保存索引字段加权因子的元数据，.nvd文件保存索引字段加权数据 |
| Per-Document Values | .dvd，.dvm | .dvm文件保存索引文档评分因子的元数据，.dvd文件保存索引文档评分数据 |
| Term Vector Index | .tvx | 将偏移存储到文档数据文件中 |
| Term Vector Documents | .tvd | 包含有term vectors的每个文档信息 |
| Term Vector Fields | .tvf | 字段级别有关term vectors的信息 |
| Live Documents | .liv | 哪些是有效文件的信息 |
| Point values | .dii，.dim | 保留索引点，如果有的话 |

#### 锁文件
默认情况下，存储在索引目录中的锁文件名为“write.lock”。如果锁目录与索引目录不同，则锁文件将命名为“XXXX-write.lock”，其中XXXX是从索引目录的完整路径导出的唯一前缀。此锁文件确保每次只有一个写入程序在修改索引。

英文文档参见：[https://lucene.apache.org/core/6_0_0/core/org/apache/lucene/codecs/lucene60/package-summary.html](https://lucene.apache.org/core/6_0_0/core/org/apache/lucene/codecs/lucene60/package-summary.html)