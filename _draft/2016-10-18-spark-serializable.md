---
layout: post
title: Spark与序列化
excerpt: Spark作业提交运行，序列化以及调优
modified: 2016-10-18
tags: [Spark, Serializable, RDD, 作业提交, 调试]
comments: true
categories: tech
author: 邓志华
---

**Spark作业提交及序列化**

  在分析Spark提交作业之前，先简单梳理一下有关Java序列化的知识以及一些理解。
  
***Java序列化***

  一个Java对象在其生命周期内，可能会有多种不同的状态，这些状态区分于同一个类的其它对象。序列化的工作，就是将对象的状态，以二进制的形态输出出来，在遵循输出协议的前提下，可以通过这些字节恢复对象在某一时刻的状态(反序列化), 这对于快速恢复以及跨JVM信息交流带来巨大的便利性。
  对象与对象之间存在着引用与被引用的关系，针对于对象本身，则可能会有不同的迭代版本。当对象尝试将对象的引用关系序列化时，需要保证引用的对象也能序列化。如下面的一个例子中，serializable(JavaSerializable) -> sc(Serializable1) ->ns(NonSerializable), 由于ns对象并不能序列化，导致在main方法中，序列化serializable操作失败。
  
<pre>
  public class JavaSerializable implements Serializable {
  Serializable1 sc;
  private JavaSerializable() { }
  public JavaSerializable(Serializable1 sc) {
    this.sc = sc;
  }
  public static void main(String[] args) throws IOException {
    JavaSerializable serializable = new JavaSerializable(new Serializable1());
    ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("/tmp/obj.ser"));
    //这里会抛出一个 "java.io.NotSerializableException: cn.creditease.basic.NonSerializable" 异常
    objectOutputStream.writeObject(serializable);
    objectOutputStream.flush();
    objectOutputStream.close();
  }
}
class Serializable1 implements Serializable {
  NonSerializable ns = new NonSerializable("hello");
}

class NonSerializable {
    int tf1;
    String tf2;
    public NonSerializable(String tf2) {
      this.tf2 = tf2;
    }
}
</pre>
然而需要认识到，序列化的工作是将对象的状态通过事先约定，输出成二进制。对于某一个类来说，如何序列化以及序列化那些状态，这些都是属于类的行为，其他的类没必要知道这些细节。因此，当要序列化的对象依赖的引用是一个null时，这个是不会序列化这个依赖对象的。例如，在上述的例子中，当serializable对象中sc的引用为NUll时，序列化serializable对象是没有问题的：

```java
public static void main(String[] args) throws IOException {
    //不初始化sc
    JavaSerializable serializable = new JavaSerializable();
    ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("/tmp/obj.ser"));
    //成功序列化serializable对象
    objectOutputStream.writeObject(serializable);
  }

```
通过反序列化恢复一个对象时，这个过程并不会调用类的构造函数。
有关更过序列化的知识，可以参考：
  http://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html

随着分布式系统的发展和流程，很多框架采用序列化的机制来实现进程之间通信，中间结果保存以及进程中一些状态的持久化。如[storm](http://storm.apache.org/)中的worker会周期性的将一些状态信息写入到指定的目录中，Supervisor反序列化这些信息，得到worker的运行状态；hadoop MapReduce会将Mapper输出的中间结果序列化成二进制文件；thrift会将客户端的调用序列化(包括方法名，以及方法中包含的参数信息)，将序列化后二进制流交给服务端来解析，实现远程调用等等。针对Java序列化改进的序列化框架也开始变得多起来,如[一些序列化对比和测试](https://github.com/eishay/jvm-serializers/wiki)。


***Spark与序列化***

   对于计算框架来说，从计算产生的原因的角度上看，可以分为两类：一种是基于事件驱动的计算模型，如storm；另一种是基于数据驱动的计算模型，如MapReduce。以这两种类型来分类的话， Spark是属于数据驱动的，也就是说，必须先有数据，而后才能进行一些数据操作，数据与操作之间是相互分离的。这些数据由一条条记录构成(可类比成传统关系型数据库中行的概念)，这些记录之间关系松散, 没有上下文(语义)。在对数据进行操作时，可以抽象出某些行为特征，尽管对记录的操作(细节)会千差万别，Spark定义了这些记录集(RDD), 以及针对这些记录集的一些操作(接口)，具体实现则由用户自己来实现。
   
   抽象是Spark一个很显著的特征，数据被抽象成数据集(RDD)，在每个数据集上记录了：
     
 *  A list of partitions 由那些数据子集构成
 *  A function for computing each split 子集中每条记录如何计算(计算因子）
 *  A list of dependencies on other RDDs 依赖于的数据集(parent RDD）
 *  Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned) 如何将该数据集的输出结果定位到下一个RDD中
 *  Optionally, a list of preferred locations to compute each split on (e.g. block locations foran HDFS file) 位置信息
   
 RDD中的这些特征包含了： 1， 我要计算的数据从何而来； 2，我要怎样计算源数据，以此来得到数据集； 3， 我如何将我的计算结果输出以及一些优化措施（计算与数据本地化)。计算数据本身没有上下文的优势，适用分治算法的基本思路，将数据切割成一个个相互独立的碎片，利用并行优势来加速计算。 这不仅可以缩短解决问题的时间，同时也适用于海量数据的问题。MapReduce, Spark就是这样的解决方案。
 
  综上所叙，从宏观上看，可以这样定义一个RDD:
  
      _rdd = f(g(h(...rdd0...)))_
 
 f,g,h 为计算因子，如filter，map，groupByKey等。
 
 如上，这些f,g,h函数的具体实现是我们定义的。从提交作业来看，我们所做的工作是提交一个jar文件，Spark要做的就是将jar中定义的类似于f，g，h这样的具体实现分发到分布式集群上，实现并行计算，当然包括容错措施。
 要实现这些特点，则必须回答以下一些问题：
   1, 怎样拉取要计算的数据。
   2，如何将f函数分发到合适的机器上进行计算
   3，如何计算。
以WordCount为例：

```scala
 val textFile = sc.textFile("hdfs:///tmp/spark.text")
 val counts = textFile.flatMap(line => line.split("\\s+"))
                 .map(word => (word, 1))
                 .reduceByKey(_ + _)
 counts.saveAsTextFile("hdfs:///tmp/result")
``` 

执行 val textFile ＝ sc.textFile("hdfs:///tmp/spark.text") 这条语句时，返回的是HadoopRDD， 这个HadoopRDD由，

```scala
override def getPartitions: Array[Partition] = {
    val jobConf = getJobConf()
    // add the credentials here as this can be called before SparkContext initialized
    SparkHadoopUtil.get.addCredentials(jobConf)
    val inputFormat = getInputFormat(jobConf)
    val inputSplits = inputFormat.getSplits(jobConf, minPartitions)
    val array = new Array[Partition](inputSplits.size)
    for (i <- 0 until inputSplits.size) {
      array(i) = new HadoopPartition(id, i, inputSplits(i))
    }
    array
  }
```

每个具体的RDD实例上提供了迭代器模式来封装RDD内部一条条记录的访问, 如HadoopRDD：

```scala
 val iter = new NextIterator[(K, V)] {
      val split = theSplit.asInstanceOf[HadoopPartition]
      logInfo("Input split: " + split.inputSplit)
      val jobConf = getJobConf()
      ...
      var reader: RecordReader[K, V] = null
      val inputFormat = getInputFormat(jobConf)
      //得到该RDD实例要处理的InputSplit，根据这个InputSplit以及InputFormat得到将内容解析成一条条<key value>的reader
      reader = inputFormat.getRecordReader(split.inputSplit.value, jobConf, Reporter.NULL)
     
      val key: K = reader.createKey()
      val value: V = reader.createValue()

      override def getNext(): (K, V) = {
        try {
          finished = !reader.next(key, value)
        } catch {
          case eof: EOFException =>
            finished = true
        }
        if (!finished) {
          inputMetrics.incRecordsRead(1)
        }
        (key, value)
      }

  }
   new InterruptibleIterator[(K, V)](context, iter)
```
 _val textFile ＝ sc.textFile("hdfs:///tmp/spark.text")_ 这一步返回了HadoopRDD的定义(其实返回的是MapPartitionsRDD，HadoopRDD返回的<Long, Text>键值对更改成value.toString, 为了描述简便，省略了这一步，具体过程可查看SparkContext.textFile源码)，包括HadoopRDD包含哪些partition，以及HadoopRDD实例是怎样获得数据(compute).
当执行到这一语句时， 
  textFile.flatMap(line => line.split("\\s+"))， 返回一个MapPartitionsRDD，

```scala 
  /**
   *  Return a new RDD by first applying a function to all elements of this
   *  RDD, and then flattening the results.
   */
  def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))
  }
```

我们自定义flatMap的逻辑f: line => line.split("\\s+")，经过_val cleanF = sc.clean(f)_清洗加工(去除原f中一些无效的变量或逻辑？），当计算该MapPartitionsRDD实例中的记录时，

```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false)
  extends RDD[U](prev) {
  //在该实例中： firstParent -> HadoopRDD
  override def compute(split: Partition, context: TaskContext): Iterator[U] = {
    val iter = firstParent[T].iterator(split, context);
    val index = split.index
    f(context, index, iter)
  }
```

当MapPartitionsRDD实例调用compute方法，生成该RDD上的迭代器时，首先查看它依赖的RDD(firstParent)对应的partition上是否有数据(缓存），如果没有计算的数据，则向依赖的RDD(HadoopRDD)要，这是一个递归计算的过程。

MapPartitionsRDD在调用compute时，有个_f(context, index, iter)_, 这个函数的调用等价于：_iter(HadoopRDD迭代器).flatMap(line => line.split("\\s+")_ 

```scala
 def flatMap[B](f: A => GenTraversableOnce[B]): Iterator[B] = new AbstractIterator[B] {
    private var cur: Iterator[B] = empty
    def hasNext: Boolean =
      cur.hasNext || self.hasNext && { cur = f(self.next).toIterator; hasNext }
    def next(): B = (if (hasNext) cur else empty).next()
  }
```

可见，我们具体的业务逻辑(line => line.split("\\s+"))是在_f(self.next)_这调用的， 而self.next获得的是HadoopRDD迭代器中下一条记录。

需要注意的是，当Spark driver执行到这一步时，属于这个Application的进程集合并没有task在运行，因为RDD实例提供了一个iterator接口，其内部数据如何计算以及要计算的数据怎样被获取，这对于其它的RDD来说是不可见的，也没必要知道。 要想真正让数据动起来(rdd.action)，则需要这样：
<pre>
  while (iter.hasNext) {
     func(iter.next)
  }
 iter.hasNext -> iter1.hasNext -> iter2.hasNext -> ..... -> itern.hasNext
 iter.next -> iter1.next -> ....... -> itern.next
 </pre> 
 我们定义的函数逻辑，则是在调用这些iter1.hasNext或者iter.next的时候被执行。
 如rdd.reduce:
 
 ```scala
 def reduce(f: (T, T) => T): T = withScope {
    val cleanF = sc.clean(f)
    val reducePartition: Iterator[T] => Option[T] = iter => {
      if (iter.hasNext) {
        Some(iter.reduceLeft(cleanF))
      } else {
        None
      }
    }
    ...
  }
 ```

回到最初的问题，Spark与序列化有何关系？
由上文的分析可知，我们定义的函数逻辑被内嵌到RDD的定义中(通过构造函数), 这些逻辑随RDD一起被分发到application可用的进程上去执行，这意味着我们定义的逻辑是RDD中一个有状态的成员变量，Spark driver通过将RDD序列化，将结果封装到相应的Task中，这部分细节可在DAGScheduler.submitMissingTasks可以看到：

```scala
var taskBinary: Broadcast[Array[Byte]] = null
    try {
      // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
      // For ResultTask, serialize and broadcast (rdd, func).
      val taskBinaryBytes: Array[Byte] = stage match {
        case stage: ShuffleMapStage =>
          closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef).array()
        case stage: ResultStage =>
          closureSerializer.serialize((stage.rdd, stage.func): AnyRef).array()
      }

      taskBinary = sc.broadcast(taskBinaryBytes)
    }
    
  //...
     case stage: ShuffleMapStage =>
          partitionsToCompute.map { id =>
            val locs = taskIdToLocations(id)
            val part = stage.rdd.partitions(id)
            new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, stage.internalAccumulators)
          }

        case stage: ResultStage =>
          val job = stage.activeJob.get
          partitionsToCompute.map { id =>
            val p: Int = stage.partitions(id)
            val part = stage.rdd.partitions(p)
            val locs = taskIdToLocations(id)
            new ResultTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, id, stage.internalAccumulators)
     }
```
将我们的逻辑序列化后，通过构造函数传入到相应的Task中，TaskSetManager再将task序列化，driver将序列化结果发送到executor， executor在反序列化得到task实例，在反序列化得到rdd和依赖关系(dependency)。 
在[SPARK-7708](https://issues.apache.org/jira/browse/SPARK-7708)提到，目前并不能支持kyro的方式来序列化这些定义的函数(待验证，spark-1.5.2)，在未可替代的前提下，使用的是Java本身提供的序列化机制。因此，我们在写我们的逻辑时，需要注意遵循Java序列化提供的规则。包括，**在我们写的逻辑中包含依赖的类，如果有状态的全局变量(递归，对象相关)不能序列化，则会出现序列化问题等**， 这个可以通过_spark.closure.serializer_可配置。 同时，了解到我们定义的函数逻辑f，会分发在不同的JVM上执行，意味着可能存在一些这样的问题，这是分布式编程特别要注意的问题：

```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)
//在分布式环境下，可能每次返回的值出现不一致
println("Counter value: " + counter)
```
[Prior to execution, Spark computes the task’s closure. The closure is those variables and methods which must be visible for the executor to perform its computations on the RDD (in this case foreach()). This closure is serialized and sent to each executor.
The variables within the closure sent to each executor are now copies and thus, when counter is referenced within the foreach function, it’s no longer the counter on the driver node. There is still a counter in the memory of the driver node but this is no longer visible to the executors! The executors only see the copy from the serialized closure. Thus, the final value of counter will still be zero since all operations on counter were referencing the value within the serialized closure.](http://spark.apache.org/docs/latest/programming-guide.html#understanding-closures-a-nameclosureslinka)
另外，在spark中，使用到了序列化来保存shuffle的中间结果,减少网络传输;有时为避免同一个RDD重复计算，需要保存某个RDD上的数据，可以通过[RDD Persist API](https://spark.apache.org/docs/latest/programming-guide.html#rdd-persistence), 使用序列化来减少内存使用量(MEMORY_ONLY_SER)或者disk的占用空间(MEMORY_AND_DISK_SER)。[对于大多数应用来说，将序列化机制更改成kyro，将数据以序列化的形式来保存，可以解决大部分应用的性能问题。](https://spark.apache.org/docs/latest/tuning.html)可以通过

```scala
//switch to using Kryo by initializing your job with a SparkConf
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer").
```
来配置或修改这部分Spark的序列化机制。<br/>
本文所作的分析不过是Spark中的冰山一角，文中不当或错误之处，欢迎批评指正。

***Useful links: <br/>***
<ul>
  <li>http://jerryshao.me/architecture/2013/10/08/spark-storage-module-analysis/</li>
  <li>https://github.com/JerryLead/SparkInternals</li>
  <li>http://spark.apache.org/docs/latest/programming-guide.html</li>
  <li>https://jaceklaskowski.gitbooks.io/mastering-apache-spark/content/spark-overview.html</li>
  <li>http://blog.madhukaraphatak.com/kryo-disk-serialization-in-spark/</li>
  <li>https://ogirardot.wordpress.com/2015/01/09/changing-sparks-default-java-serialization-to-kryo/</li>
  <li>http://spark.apache.org/docs/latest/programming-guide.html#understanding-closures-a-nameclosureslinka</li>
</ul>  