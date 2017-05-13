title: Lucene 6.0 目录结构和功能模块
date: 2016-09-27 22:50:26
tags: [Lucene]
categories: Programming Notes

---

#### Lucene英文目录结构和功能模块
- **[core](https://lucene.apache.org/core/6_0_0/core/index.html): **Lucene core library
- analyzers
  * **[analyzers-common](https://lucene.apache.org/core/6_0_0/analyzers-common/index.html): **Analyzers for indexing content in different languages and domains.
  * **[analyzers-icu](https://lucene.apache.org/core/6_0_0/analyzers-icu/index.html): **Analysis integration with ICU (International Components for Unicode).
  * **[analyzers-kuromoji](https://lucene.apache.org/core/6_0_0/analyzers-kuromoji/index.html): **Japanese Morphological Analyzer
  * **[analyzers-morfologik](https://lucene.apache.org/core/6_0_0/analyzers-morfologik/index.html): **Analyzer for dictionary stemming, built-in Polish dictionary
  * **[analyzers-phonetic](https://lucene.apache.org/core/6_0_0/analyzers-phonetic/index.html): **Analyzer for indexing phonetic signatures (for sounds-alike search)
  * **[analyzers-smartcn](https://lucene.apache.org/core/6_0_0/analyzers-smartcn/index.html): **Analyzer for indexing Chinese
  * **[analyzers-stempel](https://lucene.apache.org/core/6_0_0/analyzers-stempel/index.html): **Analyzer for indexing Polish
  *  **[analyzers-uima](https://lucene.apache.org/core/6_0_0/analyzers-uima/index.html): **Analysis integration with Apache UIMA
* **[backward-codecs](https://lucene.apache.org/core/6_0_0/backward-codecs/index.html): **Codecs for older versions of Lucene.
* **[benchmark](https://lucene.apache.org/core/6_0_0/benchmark/index.html): **System for benchmarking Lucene
* **[classification](https://lucene.apache.org/core/6_0_0/classification/index.html): **Classification module for Lucene
* **[codecs](https://lucene.apache.org/core/6_0_0/codecs/index.html): **Lucene codecs and postings formats.
* **[demo](https://lucene.apache.org/core/6_0_0/demo/index.html): **Simple example code
* **[expressions](https://lucene.apache.org/core/6_0_0/expressions/index.html): **Dynamically computed values to sort/facet/search on based on a pluggable grammar.
* **[facet](https://lucene.apache.org/core/6_0_0/facet/index.html): **Faceted indexing and search capabilities
* **[grouping](https://lucene.apache.org/core/6_0_0/grouping/index.html): **Collectors for grouping search results.
* **[highlighter](https://lucene.apache.org/core/6_0_0/highlighter/index.html): **Highlights search keywords in results
* **[join](https://lucene.apache.org/core/6_0_0/join/index.html): **Index-time and Query-time joins for normalized content
* **[memory](https://lucene.apache.org/core/6_0_0/memory/index.html): **Single-document in-memory index implementation
* **[misc](https://lucene.apache.org/core/6_0_0/misc/index.html): **Index tools and other miscellaneous code
* **[queries](https://lucene.apache.org/core/6_0_0/queries/index.html): **Filters and Queries that add to core Lucene
* **[queryparser](https://lucene.apache.org/core/6_0_0/queryparser/index.html): **Query parsers and parsing framework
* **[replicator](https://lucene.apache.org/core/6_0_0/replicator/index.html): **Files replication utility
* **[sandbox](https://lucene.apache.org/core/6_0_0/sandbox/index.html): **Various third party contributions and new ideas
* **[spatial](https://lucene.apache.org/core/6_0_0/spatial/index.html): **Geospatial search
* **[spatial3d](https://lucene.apache.org/core/6_0_0/spatial3d/index.html): **3D spatial planar geometry APIs
* **[spatial-extras](https://lucene.apache.org/core/6_0_0/spatial-extras/index.html): **Geospatial search
* **[suggest](https://lucene.apache.org/core/6_0_0/suggest/index.html): **Auto-suggest and Spellchecking support
* **[test-framework](https://lucene.apache.org/core/6_0_0/test-framework/index.html): **Framework for testing Lucene-based applications

#### Lucene中文目录结构和功能模块
- **[core](https://lucene.apache.org/core/6_0_0/core/index.html): **Lucene核心类库
- analyzers
  * **[analyzers-common](https://lucene.apache.org/core/6_0_0/analyzers-common/index.html): **不同语言和领域的内容索引分析器
  * **[analyzers-icu](https://lucene.apache.org/core/6_0_0/analyzers-icu/index.html): **集成ICU的分析器
  * **[analyzers-kuromoji](https://lucene.apache.org/core/6_0_0/analyzers-kuromoji/index.html): **日文分析器
  * **[analyzers-morfologik](https://lucene.apache.org/core/6_0_0/analyzers-morfologik/index.html): **字典词干分析器，内建的波兰语字典
  * **[analyzers-phonetic](https://lucene.apache.org/core/6_0_0/analyzers-phonetic/index.html): **索引语音特征分析器（用于类声音搜索）
  * **[analyzers-smartcn](https://lucene.apache.org/core/6_0_0/analyzers-smartcn/index.html): **索引中文分析器
  * **[analyzers-stempel](https://lucene.apache.org/core/6_0_0/analyzers-stempel/index.html): **索引波兰语分析器
  *  **[analyzers-uima](https://lucene.apache.org/core/6_0_0/analyzers-uima/index.html): **集成Apache UIMA的分析器
* **[backward-codecs](https://lucene.apache.org/core/6_0_0/backward-codecs/index.html): **Lucene旧版本的编解码器
* **[benchmark](https://lucene.apache.org/core/6_0_0/benchmark/index.html): **Lucene系统基准测试
* **[classification](https://lucene.apache.org/core/6_0_0/classification/index.html): **Lucene分类器模块
* **[codecs](https://lucene.apache.org/core/6_0_0/codecs/index.html): **Lucene编解码器和postings格式
* **[demo](https://lucene.apache.org/core/6_0_0/demo/index.html): **简单代码示例
* **[expressions](https://lucene.apache.org/core/6_0_0/expressions/index.html): **基于可插拔语法的一个动态计算的值进行sort/facet/search
* **[facet](https://lucene.apache.org/core/6_0_0/facet/index.html): **切面索引和搜索功能
* **[grouping](https://lucene.apache.org/core/6_0_0/grouping/index.html): **分组搜索结果收集器
* **[highlighter](https://lucene.apache.org/core/6_0_0/highlighter/index.html): **高亮搜索结果中的关键词
* **[join](https://lucene.apache.org/core/6_0_0/join/index.html): **标准化内容时的索引和搜索连接
* **[memory](https://lucene.apache.org/core/6_0_0/memory/index.html): **单文档内存索引实现
* **[misc](https://lucene.apache.org/core/6_0_0/misc/index.html): **索引工具和其它杂项的代码
* **[queries](https://lucene.apache.org/core/6_0_0/queries/index.html): **加入Lucene核心的过滤器和查询器
* **[queryparser](https://lucene.apache.org/core/6_0_0/queryparser/index.html): **查询解析器和解析框架
* **[replicator](https://lucene.apache.org/core/6_0_0/replicator/index.html): **文件复制工具
* **[sandbox](https://lucene.apache.org/core/6_0_0/sandbox/index.html): **各种第三方贡献和新的想法
* **[spatial](https://lucene.apache.org/core/6_0_0/spatial/index.html): **地理空间搜索
* **[spatial3d](https://lucene.apache.org/core/6_0_0/spatial3d/index.html): **3D空间平面几何的APIs
* **[spatial-extras](https://lucene.apache.org/core/6_0_0/spatial-extras/index.html): **地理空间搜索
* **[suggest](https://lucene.apache.org/core/6_0_0/suggest/index.html): **自动推荐和拼写检查
* **[test-framework](https://lucene.apache.org/core/6_0_0/test-framework/index.html): **基于Lucene的应用测试框架

**Notes**：英文水平有限，翻译若有不妥之处，还请见谅。