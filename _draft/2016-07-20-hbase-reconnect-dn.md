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
log goes here
log goes here
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

### 打印出更多Debug信息
HBase是个复杂的系统，上面的分析还不足以确定具体问题点在哪，我们还需要一些其他信息。
例如：

1. `DFSClient`的id。
2. 这次读取的气势位置和终止位置是多少。
3. 从哪个IP过来的连接。
4. *TBD*

原生的DataNode打印这个问题日志的地方并没有打印这些信息，这就需要我们做一些定制化的修改。大体的代码修改如下：

``` java
...
*TBD*
...
```

找一台DataNode上线观察，并收集一段时间的log进行分析，得到了如下的输出。

```
*TBD*
```

### Debug信息的解读
从这些log里我们能得到如下的一些信息：

1. 大体上是顺序读取，从前到后，但有一些小的偏差。
2. 频度高，每个*TBD*秒就会读取一次。
3. 每次读取的大小和偏差数都是一样的，大小是*TBD*MB，偏差是`33`，即前一个读取的最末尾在后一个读取的开始位置的下33个字节。

因为频度高跨度大并且连续，所以应该是个非常巨大的scan操作，验证了之前分析的场景：批量离线分析任务对表进行扫描。
这个神奇的数字33，每次都一样，那肯定是有原因的。我们用的是比较新的HBase，HFile是v3，看过HFile v3的说明文件就知道了，33是HFile的每个block（非HDFS里的Block）的header的长度。

### 问题的大致位置
根据前面信息的解读，出问题的地方是执行scan的过程中，读取HFile的数据的时候，对于header的处理有问题，极有可能是之前读到了header，但因为某种原因在次读的时候没有找到，所以需要回过头去再读那33个字节。

分析regionserver中关于scan的执行代码，能看到scan的执行逻辑如下：
*TBD*



### Hbase相关部分代码的解读

### 重现问题并最终确定问题点

### Patch

------

## Take away



