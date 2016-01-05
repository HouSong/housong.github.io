---
layout: post
title: 编译Impala
excerpt: 本文讲述了如何用社区办的Hadoop来编译Impala，以及遇到一些问题的解决办法。
modified: 2016-01-05
tags: [大数据, Hadoop, Hive, Presto, Impala, Drill]
comments: true
categories: tech
---

因为我们使用的是社区版本的Hadoop，Cloudera发布的Impala二进制如法执行，具体表现是log会出现pb序列化的问题，逼不得已只能用社区版的Hadoop从头编译Impala。具体过程记录如下：

0. 选一个Centos7 的机器。Centos 6会因为某些库依赖导致无法编译通过。内存不要小于4G，否则会出问题。
0. `yum install python-devel cmake openssl java-1.8.0-openjdk-devel gcc gcc-c++ lzo-devel cyrus-sasl-devel java maven python pip`
2. pip install sh
1. download package, unzip and cd into it
3. `python bin/boot*.py` （自动下载编译工具）
3. 将`thirdparty`目录中的CDH版的Hadoop替换成社区的Hadoop，将`bin/impala-config.sh`中的`IMPALA_HADOOP_VERSION`替换成社区的版本号。
4. `./buildall.sh -notests -noclean`
5. After compilation, 313MB binary is generated !!

最后出来的二进制文件很大，应该是某些优化项没有打开导致的，目前还没找到。折腾了好几天也只能有这个，不具备测试条件。之后有机会了继续。