---
layout: post
title: Hbase快速重连DataNode的问题 - 发现与分析
excerpt: 前些时间发现了DN上的一系列异常的日志，知道近期才有时间和人手研究这个问题。这篇文章介绍了一下我们发现的问题，以及查找问题根源的步骤。从中有不少值得吸取的经验。
modified: 2016-07-20
tags: [Hbase, Hadoop, HDFS, Datanode, debug, 调试]
comments: true
categories: tech
---

## 问题表现
我们的系统是这样的组成：HDFS的每台DataNode机器上都有Hbase的regionserver，Hbase分成两个集群使用。
做离线计算使用的Hbase集群数量较多，经常会有MapReduce或者Spark程序来批量scan Hbase的表。
在运维的过程中，发现DataNode的log中经常会有类似于以下的大量log重复出现：
```
.
.
.
2016-07-01 05:53:29,328 INFO org.apache.hadoop.hdfs.server.datanode.VolumeScanner: VolumeScanner(/data12/hadoop/dfs, DS-086bc494-d862-470c-86e8-9cb7929985c6): Not scheduling suspect block BP-360285305-10.130.1.11-1444619256876:blk_1095475173_21737939 for rescanning, because we rescanned it recently.
2016-07-01 05:53:29,330 INFO org.apache.hadoop.hdfs.server.datanode.VolumeScanner: VolumeScanner(/data12/hadoop/dfs, DS-086bc494-d862-470c-86e8-9cb7929985c6): Not scheduling suspect block BP-360285305-10.130.1.11-1444619256876:blk_1095475173_21737939 for rescanning, because we rescanned it recently.
2016-07-01 05:53:29,334 INFO org.apache.hadoop.hdfs.server.datanode.VolumeScanner: VolumeScanner(/data12/hadoop/dfs, DS-086bc494-d862-470c-86e8-9cb7929985c6): Not scheduling suspect block BP-360285305-10.130.1.11-1444619256876:blk_1095475173_21737939 for rescanning, because we rescanned it recently.
2016-07-01 05:53:29,340 INFO org.apache.hadoop.hdfs.server.datanode.VolumeScanner: VolumeScanner(/data12/hadoop/dfs, DS-086bc494-d862-470c-86e8-9cb7929985c6): Not scheduling suspect block BP-360285305-10.130.1.11-1444619256876:blk_1095475173_21737939 for rescanning, because we rescanned it recently.
.
.
.
```
看起来好像是有文件块损坏，但查询`dmesg`并没有发现磁盘异常，找到那个Block的文件，也能正常读取。
所以这一定有什么问题，虽然不影响正确性，但重复的读取同一个文件出错肯定会影响系统性能，浪费宝贵的资源。

由于当时人手有限，这个问题就先搁置了。近期来了一个做Hadoop的同事来帮我分析和查找问题。
这下面的工作很多是他动手完成的，这里要特别感谢邓志华。

------

## 步骤

下面讲我们是如何分步确定问题并给出解决方案的。

### 查询Block所属文件
这些出现问题的Block，在日志中能找到他们的BlockID，然后在DN的文件系统里找到这些Block对应的实际文件位置。
尝试了了几个机器后，发现这些Block的物理位置并没有什么特点，基本可以排除物理磁盘损坏或者文件系统损坏的问题。
接下来需要看这些Block在HDFS里有什么规律。
HDFS有一个工具叫`oiv`（[Offline Image Viewer](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsImageViewer.html)），将HDFS的`fsimage`导成一个xml文件，然后就可以对这个文件搜索Block所属的文件。

搜索了一些Block后，发现这些Block都是HBase的HFile文件。另外一个规律是，相近时间段的出问题的Block，经常属于同一个表，而且都是那些经常需要全表扫描的table。
结合Yarn中job执行历史，当一个表有大Job执行的时候，其所属HFile更容易出现这些问题。

由于业务场景的原因，我们不能关闭表而直接读取裸的HFile，所以只能用`Hbase client`从regionserver读取数据。
到这里能基本确定，问题的根本原因跟Hbase regionserver读取HFile有关。

### 打印出更多信息
HBase是个复杂的系统，上面的分析还不足以确定具体问题点在哪，我们还需要一些其他信息。

首先，我们打开了一台DataNode的trace日志，监控了一段时间后发现了跟之前`Not scheduling suspect block`想临近的错误，如下：
```
2016-06-30 11:21:34,320 TRACE org.apache.hadoop.hdfs.server.datanode.DataNode: DatanodeRegistration(10.130.1.29:50010, datanodeUuid=f3d795cc-2b3b-43b9-90c3-e4157c031d2c, infoPort=50075, infoSecurePort=0, ipcPort=50020, storageInfo=lv=-56;cid=CID-a99b693d-6f26-48fe-ad37-9f8162f70b22;nsid=920937379;c=0):Ignoring exception while serving BP-360285305-10.130.1.11-1444619256876:blk_1105510536_31776579 to /10.130.1.21:39933
java.net.SocketException: Original Exception : java.io.IOException: Connection reset by peer
at sun.nio.ch.FileChannelImpl.transferTo0(Native Method)
at sun.nio.ch.FileChannelImpl.transferToDirectlyInternal(FileChannelImpl.java:427)
at sun.nio.ch.FileChannelImpl.transferToDirectly(FileChannelImpl.java:492)
at sun.nio.ch.FileChannelImpl.transferTo(FileChannelImpl.java:607)
at org.apache.hadoop.net.SocketOutputStream.transferToFully(SocketOutputStream.java:223)
at org.apache.hadoop.hdfs.server.datanode.BlockSender.sendPacket(BlockSender.java:579)
at org.apache.hadoop.hdfs.server.datanode.BlockSender.doSendBlock(BlockSender.java:759)
at org.apache.hadoop.hdfs.server.datanode.BlockSender.sendBlock(BlockSender.java:706)
at org.apache.hadoop.hdfs.server.datanode.DataXceiver.readBlock(DataXceiver.java:551)
at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.opReadBlock(Receiver.java:116)
at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.processOp(Receiver.java:71)
at org.apache.hadoop.hdfs.server.datanode.DataXceiver.run(DataXceiver.java:251)
at java.lang.Thread.run(Thread.java:745)
Caused by: java.io.IOException: Connection reset by peer
... 13 more
```
可以看出，是DataNode向Client发送数据的过程中，Client主动关闭了连接，而在DataNode认为旧的连接所对应的请求数据还未被完全写完，当往一条被客户端关闭的连接继续写入数据时，则抛出`IOException: Connection reset by peer`的异常，但从这些Trace log中还是缺少一些我们需要的信息，例如：

1. `DFSClient`的id。
2. 这次读取的起始位置和终止位置是多少。
3. 从哪个IP过来的连接。

原生的DataNode打印这个问题日志的地方并没有打印这些信息，这就需要我们做一些定制化的修改。大体的格式如下：

``` java
2016-07-06 16:06:44,811 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.45:33052, bytes: ［已读取的字节数］, op: ［操作］, cliID: ［client id］, offset: ［读字节起始位置］, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: ［block ID］, duration: 此次read操作时间，单位为ns
```

找一台DataNode上线观察，并收集一段时间的log进行分析，得到了如下的输出。

```
2016-07-06 16:06:44,000 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.45:33042, bytes: 2818048, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_-1218287464_1, offset: 52549632, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1107985837_34251994, duration: 1274856797
2016-07-06 16:06:44,779 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.45:33046, bytes: 2686976, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_-1218287464_1, offset: 54608896, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1107985837_34251994, duration: 778176643
2016-07-06 16:06:44,811 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.45:33052, bytes: 2621440, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_-1218287464_1, offset: 56503296, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1107985837_34251994, duration: 31802187
```

结合之前的`Not scheduling suspect block`的日志和DataNode的代码，能知道产生这些问题的浅层原因是，相同的一个Client不断的关闭连接并重连，而且是读取的相近位置。
分析Client端的`DFSInputStream`，能得知在正常情况下（没有发生DN连接失败等错误），发生这种重连的原因基本只有一条：seek的时候发生了向前的seek操作。所以我们在这里增加了日志，位置在`seek(long position)`，代码如下：
```
if(pos > targetPos) {
  DFSClient.LOG.info(dfsClient.getClientName() + " seek " + getCurrentDatanode() + " for " + getCurrentBlock() +
          ". pos: " + pos + ", targetPos: " + targetPos);
}
```

把加了日志的Client代码在一台Hbase RegionServer上线，观察一段时间，发现了如下的一些log：
```
2016-07-10 21:31:42,147 INFO  [B.defaultRpcServer.handler=22,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 111506876, targetPos: 111506843
2016-07-10 21:31:42,715 INFO  [B.defaultRpcServer.handler=25,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 113544644, targetPos: 113544611
2016-07-10 21:31:43,341 INFO  [B.defaultRpcServer.handler=27,queue=0,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 115547269, targetPos: 115547236
2016-07-10 21:31:43,950 INFO  [B.defaultRpcServer.handler=16,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 117532235, targetPos: 117532202
2016-07-10 21:31:43,988 INFO  [B.defaultRpcServer.handler=7,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 119567867, targetPos: 119567834
2016-07-10 21:31:45,228 INFO  [B.defaultRpcServer.handler=19,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 121787264, targetPos: 121787231
2016-07-10 21:31:45,254 INFO  [B.defaultRpcServer.handler=0,queue=0,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 123910032, targetPos: 123909999
2016-07-10 21:31:46,402 INFO  [B.defaultRpcServer.handler=26,queue=2,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 125905163, targetPos: 125905130
2016-07-10 21:31:47,027 INFO  [B.defaultRpcServer.handler=24,queue=0,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 128028947, targetPos: 128028914
2016-07-10 21:31:47,649 INFO  [B.defaultRpcServer.handler=10,queue=1,port=16020] hdfs.DFSClient: DFSClient_NONMAPREDUCE_1984924661_1 seek DatanodeInfoWithStorage[10.130.1.29:50010,DS-086bc494-d862-470c-86e8-9cb7929985c6,DISK] for BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143. pos: 130057763, targetPos: 130057730
```
相同时间段内的DataNode的log如下：
```
2016-07-10 21:31:34,263 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55243, bytes: 2555904, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 82832384, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 1083375057
2016-07-10 21:31:35,080 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55248, bytes: 3080192, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 84848128, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 815747374
2016-07-10 21:31:36,197 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55256, bytes: 3080192, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 88945152, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 1090853556
2016-07-10 21:31:36,223 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55263, bytes: 2293760, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 91109888, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 25682183
2016-07-10 21:31:38,693 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55272, bytes: 2949120, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 97212928, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 1184669690
2016-07-10 21:31:38,718 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55278, bytes: 2424832, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 99258880, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 25330867
2016-07-10 21:31:41,033 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55296, bytes: 2686976, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 107441152, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 30907800
2016-07-10 21:31:43,343 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55304, bytes: 2621440, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 113544192, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 625555154
2016-07-10 21:31:43,953 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55309, bytes: 2752512, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 115547136, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 608916868
2016-07-10 21:31:45,230 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55316, bytes: 2949120, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 119567360, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 1239121220
2016-07-10 21:31:47,679 DEBUG org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /10.130.1.29:50010, dest: /10.130.1.28:55338, bytes: 2490368, op: HDFS_READ, cliID: DFSClient_NONMAPREDUCE_1984924661_1, offset: 130057728, srvID: f3d795cc-2b3b-43b9-90c3-e4157c031d2c, blockid: BP-360285305-10.130.1.11-1444619256876:blk_1109360829_35627143, duration: 26523774
```

### Debug信息的解读
从这些log里我们能得到如下的一些信息：

1. 大体上是顺序读取，从前到后，但有一些小的偏差。
2. 频度高，几百毫秒就会读取一次。
3. 每次seekde偏差数都是一样的`33`，即前一个读取的最末尾在后一个读取的开始位置的下33个字节。

因为频度高跨度大并且连续，所以应该是个非常巨大的scan操作，验证了之前分析的场景：批量离线分析任务对表进行扫描。
这个神奇的数字33，每次都一样，那肯定是有原因的。我们用的是比较新的HBase，HFile是v3，看过HFile v3的说明文件就知道了，33是HFile的每个block（非HDFS里的Block）的header的长度。

另外，Client请求的offset和DataNode的实际的offset有些区别，是因为DataNode的实际的offset要用chunk进行对齐，所以会稍向前移动一点。

### 问题的大致位置
根据前面信息的解读，出问题的地方是执行scan的过程中，读取HFile的数据的时候，对于header的处理有问题，极有可能是之前读到了header，但因为某种原因再次读的时候没有找到，所以需要回过头去再读那33个字节。

分析regionserver中关于scan的执行代码，能看到scan的执行逻辑如下：
*TBD*



### Hbase相关部分代码的解读

### 重现问题并最终确定问题点

### Patch

------

## Take away



