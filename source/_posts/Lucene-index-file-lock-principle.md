title: Lucene的索引文件锁原理
date: 2016-11-23 22:24:33
tags: [Lucene]
categories: Programming Notes

---

### 环境
Lucene 6.0.0
Java "1.8.0_111"
OS Windows 7 Ultimate

### 线程安全
在Lucene中，打开一个IndexWrite之后，就会自动在索引目录中生成write.lock文件，这个文件中并不会有内容，不管是在索引打开期间还是在索引关闭之后，其大小都为0KB，并且在IndexWriter关闭之后，并不会删除该文件。如果同时打开多个IndexWriter的话，后打开的IndexWriter就会抛出`LockObtainFailedException`异常。这是个很重要的保护机制，因为若针对同一索引打开两个writer的话，会导致索引损坏。所以Lucene中的锁主要针对并发写的情况，在写的过程中并不会影响到并发读操作。总结如下：
- 任意数量的IndexReader类都可以同时打开一个索引，与其是否同属于一个JVM无关，在单个JVM中，利用资源和发挥效率的最好办法是多线程共享单个IndexReader实例。
- 一旦建立起IndexWriter对象，系统即会分配一个锁给它，该锁只有当IndexWriter对象被关闭时才会释放。
- 对于一个索引来说，一次只能打开一个IndexWriter对象，但是对IndexReader并无限制，IndexReader对象甚至可以在IndexWriter对象正在修改索引时打开。每个IndexReader对象将向索引展示自己被打开的时间点。该对象只有在IndexWriter对象提交修改或自己被重新打开后才能获知索引修改情况。
- 任意多个线程都可以共享同一个IndexReader和IndexWriter，这些类不仅是线程安全的而且是线程有好的，说其是线程有好的意思是，它们能够很好的扩展到新增线程中，因为这些类中的同步代码数并不多。

### 利用远程文件系统访问索引
一般地，我们总是希望可以实现分布式搜索，在这方面现成的有Elasticsearch和Solr可以选用。而如果基于Lucene进行开发，如何实现呢？最简单的方案是用一台专用计算机保存和修改本地索引，然后用其他计算机通过远程文件访问来搜索该索引。该方案性能会比搜索本机索引要差得多。

一个改进的策略是通过将远程文件系统挂载至各计算机会提高一点效果。最好的效果是将索引复制到各台计算机自己的文件系统，然后再进行搜索。一个注意点是在将本地文件系统同步到远程文件系统的时候，假设远程文件系统正在被搜索，此时不能立即删除正在被使用的索引文件，此问题可以通过利用索引文件的代数解决，即搜索端用上一代索引，同步系统同步这一代索引并保留上一代索引至下一次同步时删除。另一点是如果在远程文件系统上存在定时打开的索引线程怎么办呢？为了获取最新的索引状态，通常会由一个后台线程去定时打开索引切换IndexReader，那么此时如果将本地文件系统索引同步到远程文件系统的时候一样有问题，比如`segments_*`文件同步过去了，定时线程打开`segments_*`并开始去查找其它文件，但是其它文件还未同步过去。可以通过先同步其它索引文件（非`segments_*`文件），再同步`segments_*`文件，然后更改索引代数，最后再去删除除本次和上次之外的所有无效索引文件解决。

### 索引锁机制
首先抛出几个问题再来详解。Lucene是如何知道一份索引已被其它IndexWriter打开呢？也许你会说通过write.lock文件，但是该文件里并未存有任何信息，并在IndexWriter关闭之后也不会删除该文件，如何通过write.lock文件来辨别呢？另外在程序异常退出或是JVM异常关闭都会导致锁被释放，再次启动程序和JVM之后，程序都可以正常地获取锁，这又是如何实现的呢？

在Lucene中，提供了一个顶层的抽象类LockFactory，它有三个实现类和一个子抽象类FSLockFactory，而FSLockFactory有两个实现类，分别是SimpleFSLockFactory和NativeFSLockFactory，总结如下

|锁类名|描述|
|------|-----|
|SingleInstanceLockFactory|RAMDirectory的默认锁策略，为单个进程内实例准备，意味着所有的加锁操作都通过这个实例|
|VerifyingLockFactory|用来包装其它的LockFactory，以验证每次获取锁/释放锁的操作是正确的|
|NoLockFactory |完全关闭锁机制，单例模式，你需要使用INSTANCE|
|SimpleFSLockFactory|使用Java的File.createNewFile API，它的主要缺点是在JVM崩溃时会导致遗留一个write.lock文件，在下次使用IndexWriter之前必须手动清除，比较适合NFS使用|
|NativeFSLockFactory|依赖java.nio.\*进行加锁，FSDirectory的默认锁，不适合NFS使用，该类的最大好处是JVM异常退出的话，由OS负责移除write.lock，OS并不真的删除该文件，但是会释放该文件上的所有引用，确保下次可以重新获取锁|

如果选择自行实现锁机制的话，要确认该机制能正确运行。有一个简单的调试工具LockStressTest，该类可以与LockVerifyServer和VerifyingLockFactory联合使用，以确认你自己实现的锁机制能正常运行。

值的说明的是，所有的这些类中都提供了一个获取锁的方法，以及一个与之对应的静态内部类XXXLock，在XXXLock中提供了检查锁是否有效的方法ensureValid方法和close方法。close方法主要用来关闭一些流以及释放与之相关的资源。

#### 加锁源码解析
在第一次实例化IndexWriter的过程中，Lucene首先会进行加锁操作，如果加锁失败的话，是不会进行下一步操作的。一个常见的实例化代码如下
```java
IndexWriter indexWriter = new IndexWriter(FSDirectory.open(Paths.get(indexPath)), new IndexWriterConfig(new
                WhitespaceAnalyzer()));
```
打断点开始逐步调试，程序首先进入FSDirectory的open方法
```java
public static FSDirectory open(Path path) throws IOException {
    return open(path, FSLockFactory.getDefault());
}
```
在open中调用FSLockFactory的getDefault方法
```java
public static final FSLockFactory getDefault() {
    return NativeFSLockFactory.INSTANCE;
}
```
由源码也可以知道，FSDirectory使用NativeFSLockFactory作为其默认锁策略，接着判断如果是64位的JRE并且平台支持取消映射文件，则直接返回MMapDirectory
```java
public static FSDirectory open(Path path, LockFactory lockFactory) throws IOException {
  if (Constants.JRE_IS_64BIT && MMapDirectory.UNMAP_SUPPORTED) {
    return new MMapDirectory(path, lockFactory);
  } else if (Constants.WINDOWS) {
    return new SimpleFSDirectory(path, lockFactory);
  } else {
    return new NIOFSDirectory(path, lockFactory);
  }
}
```
接着会在IndexWriterConfig的父类LiveIndexWriterConfig中进行一系列的默认初始化操作，包括
```java
LiveIndexWriterConfig(Analyzer analyzer) {
    this.analyzer = analyzer;
    //默认在内存中缓存16.0M大小的文件
    ramBufferSizeMB = IndexWriterConfig.DEFAULT_RAM_BUFFER_SIZE_MB;
    //默认禁用缓存文档
    maxBufferedDocs = IndexWriterConfig.DEFAULT_MAX_BUFFERED_DOCS;
    //禁用禁用缓存删除的Terms
    maxBufferedDeleteTerms = IndexWriterConfig.DEFAULT_MAX_BUFFERED_DELETE_TERMS;
    mergedSegmentWarmer = null;
    //索引删除策略，默认只保留最后一次提交的索引，在最新的提交成功后，删除所有之前的提交
    delPolicy = new KeepOnlyLastCommitDeletionPolicy();
    commit = null;
    //默认使用复合索引文件，主要是提升性能考虑
    useCompoundFile = IndexWriterConfig.DEFAULT_USE_COMPOUND_FILE_SYSTEM;
    //索引默认打开策略，存在则追加，不存在则创建
    openMode = OpenMode.CREATE_OR_APPEND;
    //默认的相似度计算，使用BM25Similarity
