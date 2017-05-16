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
    similarity = IndexSearcher.getDefaultSimilarity();
    //并发的合并调度器，可以指定一次最多运行的线程个数和同时合并的最大数目，如果同时合并的数目超过了线程数，那么大的合并将暂停直至小的合并完成
    mergeScheduler = new ConcurrentMergeScheduler();
    //使用默认的索引链
    indexingChain = DocumentsWriterPerThread.defaultIndexingChain;
    //使用Lucene60作为默认的编解码器，主要用来对反向索引段进行编码/解码
    codec = Codec.getDefault();
    if (codec == null) {
      throw new NullPointerException();
    }
    //调试用，用于IndexWriter和SegmentInfos等
    infoStream = InfoStream.getDefault();
    //分层合并策略
    mergePolicy = new TieredMergePolicy();
    //刷新策略，基于Ram或者数量
    flushPolicy = new FlushByRamOrCountsPolicy();
    //默认禁用reader池
    readerPooling = IndexWriterConfig.DEFAULT_READER_POOLING;
    //用来控制如何将线程分配给DocumentsWriterPerThread
    indexerThreadPool = new DocumentsWriterPerThreadPool();
    //设置单个段的RAM使用的硬上限，之后将强制刷新段
    perThreadHardLimitMB = IndexWriterConfig.DEFAULT_RAM_PER_THREAD_HARD_LIMIT_MB;
  }
```
接着会在IndexWriter的构造函数中进行获取锁的操作
```java
// obtain the write.lock. If the user configured a timeout,
// we wrap with a sleeper and this might take some time.
// 忽略上面注释，obtain(long lockWaitTimeout)已经移除
writeLock = d.obtainLock(WRITE_LOCK_NAME);
```
这里面的d在64位的JRE中就是MMapDirectory的实例，该类从BaseDirectory继承得到obtainLock方法
```java
/** Holds the LockFactory instance (implements locking for this Directory instance). */
protected final LockFactory lockFactory;
@Override
public final Lock obtainLock(String name) throws IOException {
  return lockFactory.obtainLock(this, name);
}
```
真正的获取锁的操作是在LockFactory的子类中，上面已经介绍过各个子类，所以此处的lockFactory是NativeFSLockFactory的实例，而lockFactory的引用是LockFactory类，这就是常说的父类引用指向子类实例。在LockFactory中，obtainLock是一个抽象方法，在FSLockFactory中得到实现，同时FSLockFactory是NativeFSLockFactory的父类，所以接着调用FSLockFactory中的obtainLock方法
```java
@Override
public final Lock obtainLock(Directory dir, String lockName) throws IOException {
  if (!(dir instanceof FSDirectory)) {
    throw new UnsupportedOperationException(getClass().getSimpleName() + " can only be used with FSDirectory subclasses, got: " + dir);
  }
  return obtainFSLock((FSDirectory) dir, lockName);
}
```
在obtainLock中调用obtainFSLock方法，这也是个抽象方法，由NativeFSLockFactory实现，所以这时候才真的跳转到了获取锁的核心实现方法里
```java
@Override
protected Lock obtainFSLock(FSDirectory dir, String lockName) throws IOException {
    Path lockDir = dir.getDirectory();
    // Ensure that lockDir exists and is a directory.
    // note: this will fail if lockDir is a symlink
    Files.createDirectories(lockDir);
    Path lockFile = lockDir.resolve(lockName);
    try {
        Files.createFile(lockFile);
    } catch (IOException ignore) {
        // we must create the file to have a truly canonical path.
        // if it's already created, we don't care. if it cant be created, it will fail below.
    }
    // fails if the lock file does not exist
    final Path realPath = lockFile.toRealPath();
    // used as a best-effort check, to see if the underlying file has changed
    final FileTime creationTime = Files.readAttributes(realPath, BasicFileAttributes.class).creationTime();
    if (LOCK_HELD.add(realPath.toString())) {
        FileChannel channel = null;
        FileLock lock = null;
        try {
            channel = FileChannel.open(realPath, StandardOpenOption.CREATE, StandardOpenOption.WRITE);
            lock = channel.tryLock();
            if (lock != null) {
                return new NativeFSLock(lock, channel, realPath, creationTime);
            } else {
                throw new LockObtainFailedException("Lock held by another program: " + realPath);
            }
        } finally {
            if (lock == null) { // not successful - clear up and move out
                IOUtils.closeWhileHandlingException(channel); // TODO: addSuppressed
                clearLockHeld(realPath);  // clear LOCK_HELD last 
            }
        }
    } else {
        throw new LockObtainFailedException("Lock held by this virtual machine: " + realPath);
    }
}
```
首先会递归的创建多层目录，然后创建write.lock文件，如果已经存在抛java.nio.file.FileAlreadyExistsException异常，捕获该异常之后不做任何处理，因为一旦索引创建之后，write.lock文件会一直存在，即使IndexWriter已经关闭，所以不能以该文件是否存在来判断是否有多个IndexWriter被打开了，那么是根据什么来判断的呢？

接着会获取该文件创建时候的时间戳，并且将该文件的真实路径加入LOCK_HELD，LOCK_HELD的声明如下，是一个同步的HashSet集合，在第一次打开IndexWriter的时候`LOCK_HELD.add(realPath.toString())`成功，然后调用FileChannel的API去获取锁，注意这里面重点就是`channel.tryLock()`获取到的是FileLock，这是系统级别的锁，即使其它进程想打开IndexWriter的时候，它虽然能够`LOCK_HELD.add(realPath.toString())`成功，但是在`channel.tryLock()`步会加锁失败，得到null，这时候程序依然会抛`LockObtainFailedException`异常；对于多JVM去同时写同一份索引的情况同样如此。如果在同一个进程内，后面希望打开另外的IndexWriter的时候必然`LOCK_HELD.add(realPath.toString())`失败，抛`LockObtainFailedException`异常。所以如果程序或JVM崩溃，LOCK_HELD在内存中必然也失效，系统级别的锁（使用tryLock获取的文件锁）也会释放，相当于是自动解锁了，不影响下次的重新加锁操作。
```java
private static final Set<String> LOCK_HELD = Collections.synchronizedSet(new HashSet<String>());
```

#### 释放锁源码解析
释放锁是在调用`indexWriter.close()`之后，默认在关闭时提交，所以会进入shutdown方法
```java
@Override
public void close() throws IOException {
  if (config.getCommitOnClose()) {
    shutdown();
  } else {
    rollback();
  }
}
```
在shutdown中，Lucene会先判断一系列预先设置的参数，然后进行刷新操作，将所有在内存中缓存的更新刷新到Directory中，然后静静等待合并结束，合并之后会进行内部的提交操作
```java
private void shutdown() throws IOException {
    if (pendingCommit != null) {
        throw new IllegalStateException("cannot close: prepareCommit was already called with no corresponding call to " +
                "commit");
    }
    // Ensure that only one thread actually gets to do the
    // closing
    if (shouldClose(true)) {
        boolean success = false;
        try {
            if (infoStream.isEnabled("IW")) {
                infoStream.message("IW", "now flush at close");
            }
            flush(true, true);
            waitForMerges();
            commitInternal(config.getMergePolicy());
            rollbackInternal(); // ie close, since we just committed
            success = true;
        } finally {
            if (success == false) {
                // Be certain to close the index on any exception
                try {
                    rollbackInternal();
                } catch (Throwable t) {
                    // Suppress so we keep throwing original exception
                }
            }
        }
    }
}
```
在`commitInternal(config.getMergePolicy())`方法中会调用finishCommit方法，该方法会更新索引文件的代数，同时会创建段文件信息的备份，接着调用`rollbackInternal()`方法，该方法主要是确保没有提交操作在运行，这样即使有其它线程在同步文件到硬盘中，依然可以关闭。
```java
private void rollbackInternal() throws IOException {
// Make sure no commit is running, else e.g. we can close while another thread is still fsync'ing:
  synchronized(commitLock) {
    rollbackInternalNoCommit();
  }
}
```
在`rollbackInternalNoCommit`方法中，进行释放锁的操作`IOUtils.close(writeLock)`，改方法会将所有需要关闭的资源转换为一个List链表
```java
public static void close(Closeable... objects) throws IOException {
  close(Arrays.asList(objects));
}
```
然后逐个关闭该链表中持有的资源
```java
public static void close(Iterable<? extends Closeable> objects) throws IOException {
    Throwable th = null;
    for (Closeable object : objects) {
        try {
            if (object != null) {
                object.close();
            }
        } catch (Throwable t) {
            addSuppressed(th, t);
            if (th == null) {
                th = t;
            }
        }
    }
    reThrow(th);
}
```
这里的`object.close()`关闭操作调用的就是NativeFSLock中的close方法，因为object其实就是NativeFSLock的实例，由NativeFSLock的父类Lock持有引用。NativeFSLock中的close方法如下
```java
@Override
public synchronized void close() throws IOException {
    if (closed) {
        return;
    }
    // NOTE: we don't validate, as unlike SimpleFSLockFactory, we can't break others locks
    // first release the lock, then the channel
    try (FileChannel channel = this.channel;
         FileLock lock = this.lock) {
        assert lock != null;
        assert channel != null;
    } finally {
        closed = true;
        clearLockHeld(path);
    }
}
```
因为NativeFSLock是NativeFSLockFactory的静态内部类，在finally中会直接调用NativeFSLockFactory的clearLockHeld方法，该方法代码如下
```java
private static final void clearLockHeld(Path path) throws IOException {
  boolean remove = LOCK_HELD.remove(path.toString());
  if (remove == false) {
    throw new AlreadyClosedException("Lock path was cleared but never marked as held: " + path);
  }
}
```
至此又看到了熟悉的身影，LOCK_HELD一个同步的HashSet集合会移除write.lock文件的真实路径，释放锁完毕，结束。