---
layout: post
title: Sql on Hadoop 对比
excerpt: 本文对比了一些常见的Hadoop之上的SQL引擎，分析了它们的功能，并且实际运行TPC-DS的benchmark来体现出它们的性能，并最后给使用者提出出了一些建议。
modified: 2016-01-05
tags: [大数据, Hadoop, Hive, Presto, Impala, Drill]
comments: true
categories: tech
---

本次的调研目标是分析集中常见的大规模分布式Sql引擎，包括它们的对标准Sql的支持、自定义扩展的能力、scale-out的能力、对资源的消耗、可维护性等方面，并且在TPC-DS标准数据集上进行了实地测试，得到了一系列第一手数据资料。结论是，Hive on Tez是个更好的starting point，如果经过一系列的优化后依然无法满足，可以尝试Apache Drill或者Hive LLAP。

----------

## 目标
人们对于高效的计算总是渴求的，不仅希望能处理更多的数据，而且希望越简单越好，越快越好。Hadoop出现后，开发和运维大规模分布式系统的技术越来越成熟，将传统的Sql移植到Hadoop上的需求催生出一系列的开源产品，比如[Hive](http://hive.apache.org)、[Presto](https://prestodb.io)、[Impala](http://impala.io/)、[Drill](http://drill.apache.org) 等。本文将按照以下的标准比较这几种产品，尝试给出合理的解决方案：

1. 对标准SQL的支持程度
2. 常见查询的执行性能
3. 自定义功能扩展
4. scale-out的能力
5. 载入数据的速度
5. 资源消耗
6. 可维护性
7. 特有功能

## 功能性分析
首先逐个研究产品的文档，确定各自宣称有哪些功能，一般都包括对Sql的支持程度、用户自定义函数的方式和benchmark结果等。

### Apache Hive

Hive最早来自Facebook，能解析出类似SQL的语句并转换成多个层次的MR程序序列。支持多种的文件格式，与Hadoop的生态系统完美结合，是多大规模批量查询的事实上的标准流程。经过[RC file](http://www.csdn.net/article/1970-01-01/296900)、[ORC file](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC)、[Stinger Initiative](http://hortonworks.com/blog/100x-faster-hive/)，[Stinger.next](http://zh.hortonworks.com/innovation/stinger/)等的各种发展，特别是引入[Tez](http://tez.apache.org)之后，Hive的性能、扩展性、ACID以及对事务和[SQL:2011](https://en.wikipedia.org/wiki/SQL:2011)得到了极大的发展。现在的Hive已经不是一个笨重的只会读写HDFS的简单框架，而是个复杂高效的处理系统，实时交互式分析的功能也在紧密的开发中。

### Presto
Presto被设计成一个交互式查询引擎，最早由Facebook开发来解决旧版Hive执行效率的问题。它由多个运行在Hadoop机器上的worker后台进程执行实际的任务(也可以不依赖Hadoop独立运行)，由coordinator来进行管理和调度，元数据可以连接HiveMetastore、MySql、Cassandra、Kafka等，并支持不同元数据库之间的联合操作。
所有的数据都在内存中进行计算，减少磁盘IO，速度快，但是内存会被独占，无法和Hadoop共享。如果内存中无法放下数据集，也不会像Hive一样尝试磁盘而是直接fail。
目前社区活跃，国外公司比如Facebook和国内公司比如美团、京东等都在积极实践。

### Impala
Impala由Cloudera主导，用C++来开发，直接读取HDFS的本地文件，并在自己的后台服务进程中进行解释执行。采用了与Presto相似的架构设计，impalad负责实际的任务执行和数据读写，Impala State Store负责协调impalad的实例，用户可以通过Impala Shell或者JDBC/ODBC等提交和查看任务。
与Presto不同的是，impalad只能部署在HDFS的datanode之上并且只能使用Hive的meta-store，可以使用Avro, RCFile, SequenceFile和Parquet，但不能使用ORCFile。
Impala的自定义函数可以用C++编写，也可以直接使用Hive的UDF和UDAF，但不支持UDTF。

*由于编译和安装过程太过复杂，依赖繁琐，目前还没有解决，故本次调研不包括Impala的性能测试。*

### 对比总结


| 比较项  | Hive  | Presto | Impala |
|-------:|-------|--------|--------| 
|对标准SQL的支持程度|Hive-QL，支持大部分SQL语法，即将支持SQL:2011|支持SQL-92|支持SQL-92|
|常见查询的执行性能 |参见下文|参见下文|宣称比Hive-on-tez快5到10倍|
|自定义功能扩展    |支持UDF/UDAF/UDTF|支持UDF/UDAF，与Hive不兼容|支持用C++写UDF/UDAF，也可以直接用Hive的UDF/UDAF|
|scale-out的能力 |与Hadoop集群同步增长|小范围内近似线性增长，需要单独部署|同Presto|
|载入数据的速度|支持离线批量导入和在线streaming|可以直接读Hive的实时数据和源数据库的原始数据|可以读Hive的静态数据，无法读实时数据|
|资源消耗 |由Yarn统一管理，与其他批量任务共享资源|需要独立资源分配和管理，无法与其他任务共享|同Presto|
|可维护性 |社区活跃，开发和运维人员丰富，离线批量任务的事实标准|社区活跃，有不少公司在使用|社区不是很活跃，国内有明略数据等在用，互联网企业不多|
|特有功能 |可以用Flume和Storm实时写数据|同时查询多个数据源并进行联合操作| |

## 数据集和样例查询
测试数据采用[TPC-DS](http://www.tpc.org/tpcds/)，是一个模拟商品销售记录的数据仓库，有工具来生成所需大小的文件，包含了一系列的查询，是测试数据仓库和BI分析能力的工业标准。常用的查询包括3大类：交互式的ad-hoc查询，复杂报表任务以及数据挖掘。考虑到时间有限，本文采用了如下的4个查询并提供简单介绍（详细的SQL语句参见TPC-DS的工具包）：

1. Query 3: 生产商ID是436的商品在12月份的销售额，按照年份和销售额降序排列。属于交互式查询。
2. Query 55: 2001年12月份ID是36的经理的销售额统计，按照品牌做分组，按照每个分组的销售额做降序排列。属于交互式查询。
3. Query 58: 统计在1998年8月4日那一周内3个销售渠道相同商品的销售额，筛选出在3个渠道的销售额差别在0.9到1.1之间的这些商品，按照商品编号和其中一个渠道的销售额排序，并显示出商品编号，该商品每个渠道的销售额，该商品的每个渠道销售额与平均销售额的差别，以及该商品的平均销售额。属于复杂报表任务。
4. Query 73: 在1998、1999和2000年，在给定的4个县内潜在购买能力在1000到10000并且人均拥有车辆大于1的家庭中，在每个月的第1和第2天购买次数在1到5次之间的这些人，按照购买次数排序输出客户的个人信息以及他们各自的ticket_number。属于数据挖掘任务。

为了验证各种SQL产品在不同数据规模上的表现，本文选用了3个大小的数据集，原始大小分别是100G，1T和10T，导成ORCFile后缩小到原始大小的1/4。

## 测试结果和数据解读

测试用到的软件版本：

1. Hive：社区版 1.2.1
2. Tez：社区版 0.7.0
3. Presto：0.131
4. TPC-DS：1.4.0 （[hortonworks版](https://github.com/hortonworks/hive-testbench)）

测试的过程中没有使用很复杂的手动优化和参数调整，为的是体现出产品本身的优化能力。同一个query连续执行5次取平均值，这样既能看到平均性能，也能体现出产品使用缓存的能力。
详细的结果如下：

|Hive on Tez|100G|1T|10T|
|-------:|------:|-------:|--------:|
|Query 3|51s |2m 16s |15m 13s |
|query 55| 29s |1m 27s |5m 41s |
|query 58 |41s |14m 3s |> 2hour|
|query 73|30s|1m 3s|5m 2s |

|Presto|100G|1T|10T|
|-------:|------:|-------:|--------:|
|Query 3|33s|资源不足|资源不足|
|query 55|34s|资源不足|资源不足|
|query 58 |2m 13s|20m 53s|46m 12s|
|query 73|23s|4m 4s|超时|

注：

1. Hive在10T数据量的情况下，query58跑了超过2个小时，考虑到时间有限没有让其跑完。
2. 在数据量超过1T的时候，Presto在运行query3和query55的时候提示内存不够用。
3. 在数据量10T的时候，Presto跑query73一直提示worker超时无法完成任务，原因未知。

从这些数据中我们能得到一些初步的结论：在数据量小的情况下Presto能在稍快的情况下完成，但相对Hive并没有绝对优势。在数据量变大的情况下，Presto对某些查询可能有优势（query 58），但大多数情况下无法运行或者变慢很多。对Hive而言，不管是数据大小和查询类型，都有不错的表现，但仍未达到一秒以下的响应时间。对于query58在10T的时候为什么会恶化这么多目前还不知道，不过应该可以通过优化查询来改善。

## 初步结论

Hive新版的改善明显，标准SQL的较完整支持、权限机制、JDBC/ODBC的支持、基于cost的优化机制、执行速度提升、对流式数据输入的支持等，扩大了Hive的应用场景。同时，Hive的社区活跃，应用非常广泛，开发运维的经验多。Hive的一些格式中还支持嵌套数据等复杂功能，本文并未尝试，但能给用户提供更多的选择。
Presto的应用场景稍有限，它擅长与交互式查询，但其对于Hive的优势并不是那么明显；同时它资源的消耗较多，只能在全内存的环境下运行，并且不能与hadoop共享，增加了运行和维护的成本。
再看Impala，使用C++开发，几天的尝试后依然无法编译成功。如果接下来能编译通过并顺利运行，其运维和优化成本也会很高，有丰富的生产系统C++开发经验的公司可以尝试。

所以，本文的结论是，先使用最新版的Hive，在遇到瓶颈的时候先尝试参数调优和SQL语句优化。对于亚秒级的查询，如果需求非常迫切，可以尝试[Apache Drill](http://drill.apache.org/)，或者等待[Hive LLAP](https://issues.apache.org/jira/browse/HIVE-7926)的完成。
