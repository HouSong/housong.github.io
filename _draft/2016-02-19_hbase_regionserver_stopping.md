---
layout: post
title: Hbase regionserver 无法正常停止的问题分析
excerpt: 有几次在关闭Hbase regionserver的时候发现无法停止，卡在kill之后。进行一些分析之后，发现了一些问题。这篇文章进行简要的介绍。
modified: 2016-02-19
tags: [Hbase, jstack, 代码分析]
comments: true
categories: tech
---

## 问题现象
尝试停止region server的时候，发现长时间卡在打点的地方没有反应，后续的手动kill也都没有反应，只有`kill -9`才能结束进程。
看log也没有发现很可疑的地方。
我的region server都是用daemon-tools来管理的，进程出错退出时能自动重启，但这个错误将导致进程重启失效。
于是开始着手研究。

## 分析方法
停止
jstack log:
```
"regionserver/c1-hd-dn2.bdp.idc/10.130.1.21:16020" #22 prio=5 os_prio=0 tid=0x00007f12b57a7000 nid=0x14a15 waiting on condition [0x00007f1279c8d000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at org.apache.hadoop.hbase.ipc.RpcClientImpl.close(RpcClientImpl.java:1170)
        at org.apache.hadoop.hbase.regionserver.HRegionServer.run(HRegionServer.java:1061)
        at java.lang.Thread.run(Thread.java:745)

"Thread-5" #38 prio=5 os_prio=0 tid=0x000000000428f000 nid=0x1a9b0 in Object.wait() [0x00007f10afdfd000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        at java.lang.Thread.join(Thread.java:1245)
        - locked <0x000000051d99bae0> (a java.lang.Thread)
        at org.apache.hadoop.hbase.util.Threads.shutdown(Threads.java:111)
        at org.apache.hadoop.hbase.util.Threads.shutdown(Threads.java:99)
        at org.apache.hadoop.hbase.regionserver.ShutdownHook$ShutdownHookThread.run(ShutdownHook.java:115)
        at org.apache.hadoop.util.ShutdownHookManager$1.run(ShutdownHookManager.java:54)

"RS_LOG_REPLAY_OPS-c1-hd-dn2:16020-1" #63042 prio=5 os_prio=0 tid=0x00007f12b43e8800 nid=0x151e8 waiting on condition [0x00007f10aecec000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000051e0d79d0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"RS_CLOSE_REGION-c1-hd-dn2:16020-2" #1954 prio=5 os_prio=0 tid=0x00007f129917e800 nid=0x18b4b waiting on condition [0x00007f10af0f0000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000051e0d77d8> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"RS_OPEN_REGION-c1-hd-dn2:16020-2" #140 prio=5 os_prio=0 tid=0x00007f12b57fd000 nid=0x14c85 waiting on condition [0x00007f10b6965000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000051ded2b50> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

"main" #1 prio=5 os_prio=0 tid=0x00007f12b4012800 nid=0x1498c in Object.wait() [0x00007f12bc4a2000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        at java.lang.Thread.join(Thread.java:1245)
        - locked <0x000000051d99bae0> (a java.lang.Thread)    # 0x000000051d99bae0 is HRegionServer
        at java.lang.Thread.join(Thread.java:1319)
        at org.apache.hadoop.hbase.util.HasThread.join(HasThread.java:89)
        at org.apache.hadoop.hbase.regionserver.HRegionServerCommandLine.start(HRegionServerCommandLine.java:66)
        at org.apache.hadoop.hbase.regionserver.HRegionServerCommandLine.run(HRegionServerCommandLine.java:87)
        at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
        at org.apache.hadoop.hbase.util.ServerCommandLine.doMain(ServerCommandLine.java:126)
        at org.apache.hadoop.hbase.regionserver.HRegionServer.main(HRegionServer.java:2651)

```

## 原因
`kill`进程后，`org.apache.hadoop.util.ShutdownHookManager`开始执行，当要关闭`HRegionServer`的时候卡在了`HRegionServer.java:1061`行，在关闭一个`RpcClient`。由于ShutdownHookManager是顺序执行的，这个卡住就导致其他的hook无法执行。这个Rpc Client是用于状态汇报的stub，它在close的过程中卡在了一个循环等待中，创建的connections一直没有完全停止（RpcClientImpl.java的1168行到1177行）。


## 解决途径
