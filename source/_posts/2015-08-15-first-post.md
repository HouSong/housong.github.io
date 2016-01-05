---
layout: post
title: 大规模社会化数据--概念与实现（0）
excerpt: 在这个系列文章中，我将一步步的分享我在大规模社会化数据的实践中获得的经验。这篇文章将简要介绍我在做什么，以及这个东西有什么用。
modified: 2015-08-15
tags: [知识图谱, 大数据, 风控, Hadoop, HBase, Elastic Search, Titan Graph]
comments: true
categories: tech
---

本文将简要介绍人们对于数据管理的演化，大规模社会化的数据给已有技术带来的挑战，以及我们如何能很好的解决这些挑战同时又为未来做好准备。

## 回望历史
人们一直在寻找管理知识和使用知识的更高效的方式，但受限于当时的技术条件，人们有时无法解决已有的问题，当然也没有能力和胆量来想象更复杂的世界。当人们还只知道结绳记事的适合，他们无法想象能用文字把事情的细节描述的如此清晰。当印刷术还不完善的时候，文字还只是极少部分人的特权，大部分人活在愚昧之中。阿兰图灵设计的密码破译机[Bombe](https://en.wikipedia.org/wiki/Bombe)以机器的运算代替密码学家们的手动分析，战胜了强大的[Enigma密码机](https://en.wikipedia.org/wiki/Enigma_machine)，使得纳粹的加密电文显露无疑，为二战中盟军的最后胜利提供了强有力的支持。这些看似和数据管理无关的历史，其实都是人们对信息的加工，如何能更准确、更高效、更安全、更方便的共享知识，产生新的知识和财富。

现代的计算机工业出现后，出现了很多的高效的数据管理技术，不得不提的是各种类型的数据库管理系统。[面向对象的数据库(OODBMS)](https://en.wikipedia.org/wiki/Object_database)仿照面向对象的编程技术，使得其可以直接被C++, C#, Java等面向对象编程语言直接使用，但因为其复杂性变得难以优化。[关系型数据库(RDMBS)](https://en.wikipedia.org/wiki/Relational_database_management_system)因其简洁的结构和强有力的数学基础，很快赢得了商业上的成功。SQL语句给RDBMS提供了接近自然语言的接口，使在线的随机OLTP任务和离线的OLAP分析任务可以统一在一个框架下。现代的RDBMS系统提供了更多的高级功能，比如安全性、事务操作、高可用等，成为了计算机软件开发者和数据分析科学家的必备技能。

互联网时代的来临，给人们带来更便捷的生活的同时，也给数据的加工和处理技术带来了更大的挑战，大数据技术也从学术界的畅想变为工业界的现实。人们在网络上的一举一动，在大数据的技术下，已经不是什么隐私，而变成了网络服务提供者的资产。你在购物网站上随意点击的几个商品，会被电商网站抽取出关键路径，然后给你推荐商品。骗子在网上发布的欺诈帖子，会作为他一生的污点记录在各种搜索引擎、骗子数据库或者网贷平台中。主要技术包括[GFS](https://en.wikipedia.org/wiki/Google_File_System)和[HDFS](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html)一类的分布式文件系统，[BigTable](https://en.wikipedia.org/wiki/BigTable)、[HBase](http://hbase.apache.org)和[Cassandra](http://cassandra.apache.org)一类的NoSql存储，[MapReduce](https://en.wikipedia.org/wiki/MapReduce)、[Spark](http://spark.apache.org)和[Tez](https://tez.apache.org/)一类的计算模型，和[Mahout](http://mahout.apache.org)和[MLlib](http://spark.apache.org/mllib/)一类的机器学习算法库，等等。无数的顶尖工程师和科学家的努力，让童话变成现实，改变着我们的每一天的生活。

## 现在的挑战

我们在以前所未有的速度创造和记录着各种各样的数据，而我们对于数据分析和挖掘的胃口也史无前例的飞速增长着。大数据是个热词，各种人都在谈论着，计算机从业人纷纷变成了大数据专家，猎头们撒开大网，打爆每个标着“大数据”的简历。大数据是个好东西，但它不是万能药，需要对症下药。

大数据技术的能力，可以总结为4个V，Volume，Velocity，Variety和Value。现有的大数据技术，以Volume（大的数据量）为开始，逐步加入了velocity（时效性）考量。Value（价值）则体现在两个方面，一是原始数据的价值，1TB的信用卡账单和1TB的网站点击日志，其价值自然相差很多；二是，需要从这些数据中得到价值，不能仅仅为了好看的技术而白费力气。

现在的数据类别越来越多、格式越来越庞杂，但大数据技术对于Variety（多样性）的支持似乎并没有开始。<!--例子，价值在哪，为什么不好做-->

## 站在巨人的肩膀上
*已经有了哪些技术我们能用？还缺点什么*

## 乘风远航
*我们的目标是什么？有什么需要克服的？有什么可以畅想？*
