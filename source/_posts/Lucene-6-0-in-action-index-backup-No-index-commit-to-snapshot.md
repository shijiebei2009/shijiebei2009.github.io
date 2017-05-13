title: Lucene 6.0 实战-索引备份No index commit to snapshot
date: 2016-06-24 23:16:03
tags: [Lucene]
categories: Programming Notes

---

###备份之前先提交

在使用Lucene进行索引备份的时候，报错
>Exception in thread "main" java.lang.IllegalStateException: No index commit to snapshot
at org.apache.lucene.index.SnapshotDeletionPolicy.snapshot(SnapshotDeletionPolicy.java:160)

其中导致抛错的代码是
```java
SnapshotDeletionPolicy snapshotDeletionPolicy = (SnapshotDeletionPolicy)
indexWriter.getConfig().getIndexDeletionPolicy();
IndexCommit snapshot = snapshotDeletionPolicy.snapshot();
```
查看snapshot()方法源码发现
```java
if (lastCommit == null) {
  // No commit yet, eg this is a new IndexWriter:
  throw new IllegalStateException("No index commit to snapshot");
}
```
可见如果是第一次也就是说还没有commit过，这种情况通常是第一次使用IndexWriter的时候，那么就会抛此异常。解决办法很简单，就是在热备之前commit索引，将所有的缓存flush到硬盘，这也是热备的正确逻辑，可以确保将索引的最新变动保存到备份之中。

###确保使用同一个SnapshotDeletionPolicy

另外还有一点需要注意，如果你在备份的时候使用的是新new的SnapshotDeletionPolicy，那么同样会抛出此异常
```java
SnapshotDeletionPolicy snapshot= new SnapshotDeletionPolicy(new KeepOnlyLastCommitDeletionPolicy());
IndexCommit commit = snapshot.snapshot();
Collection<String> fileNames = commit.getFileNames();
```
这是因为采用new的形式得到的SnapshotDeletionPolicy和最初实例化IndexWriter的时候使用的并不是同一个，所以对于IndexWriter所做的提交，新的SnapshotDeletionPolicy是无法知道的。正确方式如下
```java
IndexWriterConfig config = (IndexWriterConfig) indexWriter.getConfig();
SnapshotDeletionPolicy snapshotDeletionPolicy = (SnapshotDeletionPolicy) 
config.getIndexDeletionPolicy();
IndexCommit snapshot = snapshotDeletionPolicy.snapshot();
//设置索引提交点，默认是null，会打开最后一次提交的索引点
config.setIndexCommit(snapshot);
Collection<String> fileNames = snapshot.getFileNames();
```