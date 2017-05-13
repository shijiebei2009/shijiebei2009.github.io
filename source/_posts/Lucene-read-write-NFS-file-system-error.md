title: Lucene读写NFS文件系统异常
date: 2016-08-12 22:47:34
tags: [Lucene]
categories: Programming Notes

---

>Caused by: java.lang.InternalError: a fault occurred in a recent unsafe memory access operation in compiled Java code
at org.apache.lucene.store.DataInput.readVInt(DataInput.java:134) ~[lucene-core-5.5.0.jar:5.5.0 2a228b3920a07f930f7afb6a42d0d20e184a943c - mike - 2016-02-16 15:18:34]
at org.apache.lucene.codecs.blocktree.SegmentTermsEnumFrame.loadBlock(SegmentTermsEnumFrame.java:157) ~[lucene-core-5.5.0.jar:5.5.0 2a228b3920a07f930f7afb6a42d0d20e184a943c - mike - 2016-02-16 15:18:34]
at org.apache.lucene.codecs.blocktree.SegmentTermsEnum.seekExact(SegmentTermsEnum.java:507) ~[lucene-core-5.5.0.jar:5.5.0 2a228b3920a07f930f7afb6a42d0d20e184a943c - mike - 2016-02-16 15:18:34]
at org.apache.lucene.index.LeafReader.docFreq(LeafReader.java:152) ~[lucene-core-5.5.0.jar:5.5.0 2a228b3920a07f930f7afb6a42d0d20e184a943c - mike - 2016-02-16 15:18:34]
at org.apache.lucene.search.IndexSearcher.count(IndexSearcher.java:394) ~[lucene-core-5.5.0.jar:5.5.0 2a228b3920a07f930f7afb6a42d0d20e184a943c - mike - 2016-02-16 15:18:34]

通常来说，在NFS上大规模的操作总是容易出问题的。如果你使用的是NFS（Network File System）那么建议你升级NFS文件系统所在机器的JDK到Java的最新版，而不是升级你跑程序的机器，因为NFS文件系统所在的服务器很可能和你跑程序的不在同一台服务器，通常情况下都是将NFS作为一块硬盘挂载到某个服务器之上。

另外Lucene本身对NFS的支持就不够好，所以如果你服务器资源充足的话，建议在一个服务器上跑写索引进程，在另一台服务器上跑读索引进程，这样也能避免此类问题。

另外，[unsafe memory access operation](http://www.gossamer-threads.com/lists/lucene/java-user/253368)这个帖子中碰到了同样的问题，可以参考之。