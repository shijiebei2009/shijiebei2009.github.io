title: Lucene 6.0 实战（3）-各种Field查询操作
date: 2016-05-20 21:33:32
tags: [Lucene]
categories: Programming Notes

---

###IntPoint查询
XXXPoint类中提供了一些常用的静态工厂查询方法，可以直接用来构建查询语句。
```java
@Test
public void testIntPointQuery() throws IOException {
    Directory directory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(directory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    Field intPoint = new IntPoint("age", 11);
    document.add(intPoint);
    intPoint = new StoredField("age", 11);
    document.add(intPoint);
    indexWriter.addDocument(document);
    Field intPoint1 = new IntPoint("age", 22);
    document = new Document();
    document.add(intPoint1);
    intPoint1 = new StoredField("age", 22);
    document.add(intPoint1);
    indexWriter.addDocument(document);
    indexWriter.close();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(directory));
    //精确查询
    Query query = IntPoint.newExactQuery("age", 11);
    ScoreDoc[] scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("精确查询：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，不包含边界
    query = IntPoint.newRangeQuery("age", Math.addExact(11, 1), Math.addExact(22, -1));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("不包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，包含边界
    query = IntPoint.newRangeQuery("age", 11, 22);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，左包含，右不包含
    query = IntPoint.newRangeQuery("age", 11, Math.addExact(22, -1));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("左包含右不包含：" + indexSearcher.doc(scoreDoc.doc));
    }
    //集合查询
    query = IntPoint.newSetQuery("age", 11, 22, 33);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("集合查询：" + indexSearcher.doc(scoreDoc.doc));
    }
}
```
输出结果如下
```java
精确查询：Document<stored<age:11>>
包含边界：Document<stored<age:11>>
包含边界：Document<stored<age:22>>
左包含右不包含：Document<stored<age:11>>
集合查询：Document<stored<age:11>>
集合查询：Document<stored<age:22>>
```

###LongPoint查询
```java
@Test
public void testLongPointQuery() throws IOException {
    Directory directory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(directory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    Field longPoint = new LongPoint("age", 11);
    document.add(longPoint);
    longPoint = new StoredField("age", 11);
    document.add(longPoint);
    indexWriter.addDocument(document);
    longPoint = new LongPoint("age", 22);
    document = new Document();
    document.add(longPoint);
    longPoint = new StoredField("age", 22);
    document.add(longPoint);
    indexWriter.addDocument(document);
    indexWriter.close();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(directory));
    //精确查询
    Query query = LongPoint.newExactQuery("age", 11);
    ScoreDoc[] scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("精确查询：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，不包含边界
    query = LongPoint.newRangeQuery("age", Math.addExact(11, 1), Math.addExact(22, -1));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("不包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，包含边界
    query = LongPoint.newRangeQuery("age", 11, 22);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，左包含，右不包含
    query = LongPoint.newRangeQuery("age", 11, Math.addExact(22, -1));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("左包含右不包含：" + indexSearcher.doc(scoreDoc.doc));
    }
    //集合查询
    query = LongPoint.newSetQuery("age", 11, 22, 33);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("集合查询：" + indexSearcher.doc(scoreDoc.doc));
    }
}
```
输出结果如下
```java
精确查询：Document<stored<age:11>>
包含边界：Document<stored<age:11>>
包含边界：Document<stored<age:22>>
左包含右不包含：Document<stored<age:11>>
集合查询：Document<stored<age:11>>
集合查询：Document<stored<age:22>>
```

###FloatPoint查询
```java
@Test
public void testFloatPointQuery() throws IOException {
    Directory directory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(directory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    Field floatPoint = new FloatPoint("age", 11.1f);
    document.add(floatPoint);
    floatPoint = new StoredField("age", 11.1f);
    document.add(floatPoint);
    indexWriter.addDocument(document);
    floatPoint = new FloatPoint("age", 22.2f);
    document = new Document();
    document.add(floatPoint);
    floatPoint = new StoredField("age", 22.2f);
    document.add(floatPoint);
    indexWriter.addDocument(document);
    indexWriter.close();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(directory));
    //精确查询
    Query query = FloatPoint.newExactQuery("age", 11.1f);
    ScoreDoc[] scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("精确查询：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，不包含边界
    query = FloatPoint.newRangeQuery("age", Math.nextUp(11.1f), Math.nextDown(22.2f));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("不包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，包含边界
    query = FloatPoint.newRangeQuery("age", 11.1f, 22.2f);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，左包含，右不包含
    query = FloatPoint.newRangeQuery("age", 11.1f, Math.nextDown(22.2f));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("左包含右不包含：" + indexSearcher.doc(scoreDoc.doc));
    }
    //集合查询
    query = FloatPoint.newSetQuery("age", 11.1f, 22.2f, 33.3f);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("集合查询：" + indexSearcher.doc(scoreDoc.doc));
    }
}
```
输出结果如下
```java
精确查询：Document<stored<age:11.1>>
包含边界：Document<stored<age:11.1>>
包含边界：Document<stored<age:22.2>>
左包含右不包含：Document<stored<age:11.1>>
集合查询：Document<stored<age:11.1>>
集合查询：Document<stored<age:22.2>>
```

###DoublePoint查询
```java
@Test
public void testDoublePointQuery() throws IOException {
    Directory directory = new RAMDirectory();
    IndexWriter indexWriter = new IndexWriter(directory, new IndexWriterConfig(new StandardAnalyzer()));
    Document document = new Document();
    Field doublePoint = new DoublePoint("age", 11.1);
    document.add(doublePoint);
    doublePoint = new StoredField("age", 11.1);
    document.add(doublePoint);
    indexWriter.addDocument(document);
    doublePoint = new DoublePoint("age", 22.2);
    document = new Document();
    document.add(doublePoint);
    doublePoint = new StoredField("age", 22.2);
    document.add(doublePoint);
    indexWriter.addDocument(document);
    indexWriter.close();
    IndexSearcher indexSearcher = new IndexSearcher(DirectoryReader.open(directory));
    //精确查询
    Query query = DoublePoint.newExactQuery("age", 11.1);
    ScoreDoc[] scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("精确查询：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，不包含边界
    query = DoublePoint.newRangeQuery("age", Math.nextUp(11.1), Math.nextDown(22.2));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("不包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，包含边界
    query = DoublePoint.newRangeQuery("age", 11.1, 22.2);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("包含边界：" + indexSearcher.doc(scoreDoc.doc));
    }
    //范围查询，左包含，右不包含
    query = DoublePoint.newRangeQuery("age", 11.1, Math.nextDown(22.2));
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("左包含右不包含：" + indexSearcher.doc(scoreDoc.doc));
    }
    //集合查询
    query = DoublePoint.newSetQuery("age", 11.1, 22.2, 33.3);
    scoreDocs = indexSearcher.search(query, 10).scoreDocs;
    for (ScoreDoc scoreDoc : scoreDocs) {
        System.out.println("集合查询：" + indexSearcher.doc(scoreDoc.doc));
    }
}
```
输出结果如下
```java
精确查询：Document<stored<age:11.1>>
包含边界：Document<stored<age:11.1>>
包含边界：Document<stored<age:22.2>>
左包含右不包含：Document<stored<age:11.1>>
集合查询：Document<stored<age:11.1>>
集合查询：Document<stored<age:22.2>>
```