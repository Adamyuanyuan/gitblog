---
title: Spark源码阅读之——StreamingContext详解
toc: true
date: 2017-01-03 14:23:04
tags: spark streaming
categories: spark
---

[Spark Streaming 源码解析系列](https://github.com/lw-lin/CoolplaySpark/tree/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97)很好地解析了Spark Streaming框架的源码，遗留了一点关于StreamingContext的解析，我基于自己的理解，简要阐述如下：


```
本系列内容适用范围：
* 2016.12.28 update, Spark 2.1 全系列 √ (2.1.0)
* 2016.11.14 update, Spark 2.0 全系列 √ (2.0.0, 2.0.1, 2.0.2)
* 2016.11.07 update, Spark 1.6 全系列 √ (1.6.0, 1.6.1, 1.6.2, 1.6.3)
```

阅读本文前，请一定先阅读 [Spark Streaming 实现思路与模块概述](0.1 Spark Streaming 实现思路与模块概述.md) 一文，其中概述了 Spark Streaming 的 4 大模块的基本作用，有了全局概念后再看本文对 `StreamingContext` 细节的解释。

## 引言

![image](040.png)

如各个模块的架构图所示，`StreamingContext` 是 Spark Streaming 提供给用户 code 的、与前述 4 个模块交互的一个简单和统一的入口，是Spark Streaming程序与Spark Core的连接器，下面我们用这段11行的完整 [quick example](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)，来说明用户 code 是怎么通过 `StreamingContext` 与前面几个模块进行交互的：

```scala
import org.apache.spark._
import org.apache.spark.streaming._

// 首先配置一下本 quick example 将跑在本机，app name 是 NetworkWordCount
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
// batchDuration 设置为 1 秒，然后创建一个 streaming 入口
val ssc = new StreamingContext(conf, Seconds(1))

// ssc.socketTextStream() 将创建一个 SocketInputDStream；这个 InputDStream 的 SocketReceiver 将监听本机 9999 端口
val lines = ssc.socketTextStream("localhost", 9999)

val words = lines.flatMap(_.split(" "))      // DStream transformation
val pairs = words.map(word => (word, 1))     // DStream transformation
val wordCounts = pairs.reduceByKey(_ + _)    // DStream transformation
wordCounts.print()                           // DStream output
// 上面 4 行利用 DStream transformation 构造出了 lines -> words -> pairs -> wordCounts -> .print() 这样一个 DStreamGraph
// 但注意，到目前是定义好了产生数据的 SocketReceiver，以及一个 DStreamGraph，这些都是静态的

// 下面这行 start() 将在幕后启动 JobScheduler, 进而启动 JobGenerator 和 ReceiverTracker
// ssc.start()
//    -> JobScheduler.start()
//        -> JobGenerator.start();    开始不断生成一个一个 batch
//        -> ReceiverTracker.start(); 开始往 executor 上分布 ReceiverSupervisor 了，也会进一步创建和启动 Receiver
ssc.start()

// 然后用户 code 主线程就 block 在下面这行代码了
// block 的后果就是，后台的 JobScheduler 线程周而复始的产生一个一个 batch 而不停息
// 也就是在这里，我们前面静态定义的 DStreamGraph 的 print()，才一次一次被在 RDD 实例上调用，一次一次打印出当前 batch 的结果
ssc.awaitTermination()
```

从上述样例程序可知，程序的前两行创建了一个新的 `StreamingContext`，第三行通过 `ssc.socketTextStream`通过ssc暴露的方法创建了一个`ReceiverInputDStream`，接着基于DStream的各种方法对数据进行了操作，最后通过 `ssc.start` 方法启动了Spark Streaming 程序，最后一句`ssc.awaitTermination()`将用户 code 主线程 block 住，由后台的 JobScheduler 线程周而复始的产生一个一个 batch 而不停息地处理，除非发生异常。

我们可以发现，StreamingContext主要包含以下内容：

- StreamingContext的创建(构造函数)
- StreamingContext的初始化(成员)
- StreamingContext的状态控制(函数)


### StreamingContext的创建

首先我们来看`StreamingContext`的构造函数，主要由三个参数，分别是：
- SparkContext：SparkStreaming的最终处理是交给SparkContext的；
- Checkpoint：检查点，用于错误恢复；
- Duration：设定Streaming每个批次的积累时间。

``` scala
class StreamingContext private[streaming] (
    _sc: SparkContext,
    _cp: Checkpoint,
    _batchDur: Duration
  ) extends Logging {
```

`StreamingContext` 有以下几种不同的创建方式：

``` scala

  // 1. 通过已经存在的SparkContext创建.
  def this(sparkContext: SparkContext, batchDuration: Duration) = {
    this(sparkContext, null, batchDuration)
  }

  // 2. 通过SparkConf中的配置信息来来创建.
  def this(conf: SparkConf, batchDuration: Duration) = {
    this(StreamingContext.createNewSparkContext(conf), null, batchDuration)
  }

  // 3. 通过一些必要参数来创建
  def this(
      master: String,
      appName: String,
      batchDuration: Duration,
      sparkHome: String = null,
      jars: Seq[String] = Nil,
      environment: Map[String, String] = Map()) = {
    this(StreamingContext.createNewSparkContext(master, appName, sparkHome, jars, environment),
         null, batchDuration)
  }

  // 4. 从checkpoint文件中读取来重新创建
  def this(path: String, hadoopConf: Configuration) =
    this(null, CheckpointReader.read(path, new SparkConf(), hadoopConf).orNull, null)

  // ...
  // 5. 已经存在的SparkContext，通过读取checkpoint文件重新创建
  def this(path: String, sparkContext: SparkContext) = {
    this(
      sparkContext,
      CheckpointReader.read(path, sparkContext.conf, sparkContext.hadoopConfiguration).orNull,
      null)
  }
  // ...
  // 6. 值得一提的是，StreamingContext对象提供了一个构造方法，如果存在Checkpoint就通过Checkpoint创建，否则新建一个StreamingContext
    def getOrCreate(
      checkpointPath: String,
      creatingFunc: () => StreamingContext,
      hadoopConf: Configuration = SparkHadoopUtil.get.conf,
      createOnError: Boolean = false
    ): StreamingContext = {
    val checkpointOption = CheckpointReader.read(
      checkpointPath, new SparkConf(), hadoopConf, createOnError)
    checkpointOption.map(new StreamingContext(null, _, null)).getOrElse(creatingFunc())
  }

```

整理上述的文件创建过程，可以看出，StreamingContext的创建是一定要包含SparkContext的，同理也可以推出，Spark Streaming最终实际是交给SparkContext来处理的，Spark Streaming更像是Spark Core的一个应用程序。
`StreamingContext`的创建主要分为两类：

- 1: 通过SparkContext建立新的StreamingContext，需要指定`batchDuration`时间；
- 2: 从checkpoint文件中读取的Checkpoint对象中创建StreamingContext，用于异常情况下的恢复，`batchDuration`在Checkpoint中已经保存，所以可以不用显示指定。

所以说，要构建StreamingContext，就必须要以上两者至少选一，下面的代码也说明了这点：

``` scala
  require(_sc != null || _cp != null,
    "Spark Streaming cannot be initialized with both SparkContext and checkpoint as null")
```
注意：由于SparkStreaming至少需要一个线程来接收数据，所以local与local[1]模式下是不可以启动的。

### StreamingContext的初始化

StreamingContext在创建后会进行一些初始化（静态定义）的工作，定义一些静态的数据结构，由图可知，StreamingContext主要持有 DStreamGraph 与 JobScheduler 的对象：

如下是graph的初始化定义代码，如果之前存在 Checkpoint ，则graph从 Checkpoint 得到，否则创建一个新的graph
``` scala
  private[streaming] val graph: DStreamGraph = {
    if (isCheckpointPresent) {
      _cp.graph.setContext(this)
      // 遍历从Checkpoint数据中恢复出RDD
      _cp.graph.restoreCheckpointData()
      _cp.graph
    } else {
      require(_batchDur != null, "Batch duration for StreamingContext cannot be null")
      val newGraph = new DStreamGraph()
      newGraph.setBatchDuration(_batchDur)
      newGraph
    }
  }
```
如下是初始化jobScheduler的代码：
``` scala
  private[streaming] val scheduler = new JobScheduler(this)
```

除了上述两个主要成员，StreamingContext还包含以下成员：
- ContextWaiter：用于等待任务执行结束；
- StreamingJobProgressListener：用于监听StreamingJob，用以更新StreamingTab的显示；
- StreamingTab：用于生成SparkUI中Streaming那一页标签；
- StreamingSource： 流式计算的测量数据源metrics。

除了定义上述成员，StreamingContext还进行了Checkpoint,创建了Checkpoint目录：

``` scala
conf.getOption("spark.streaming.checkpoint.directory").foreach(checkpoint)
```
checkpoint方法如下：
``` scala
  // 创建Checkpoint目录
  def checkpoint(directory: String) {
    if (directory != null) {
      val path = new Path(directory)
      val fs = path.getFileSystem(sparkContext.hadoopConfiguration)
      fs.mkdirs(path)
      val fullPath = fs.getFileStatus(path).getPath().toString
      sc.setCheckpointDir(fullPath)
      checkpointDir = fullPath
    } else {
      checkpointDir = null
    }
  }
```

### StreamingContext的控制

StreamingContext作为控制面板，给用户提供了许多控制方法，就像控制面板上的按钮，让我们来开发spark Streaming程序，主要有以下方法：

- sparkContext： 获得ssc所属的SparkContext
- remember(duration: Duration)：通过设置 `graph.remember(duration)` 来设置`rememberDuration`

简单解释一下`rememberDuration`，Spark Streaming 会在每个Batch任务结束时进行一次清理动作`clearMetadata`，每个DStream 都会被扫描，先清理输出Dstream，接着清理输入DStream，清理的时候，根据`rememberDuration`来计算出oldRDD然后清理。`rememberDuration` 有默认值，大体是`slideDuration`，也就是DStream生成RDD的时间间隔，如果设置了checkpointDuration 则是2*checkpointDuration，手动指定的值要大于默认值才会生效。

接下来是定义一些定义输入流的方法，主要有：

- receiverStream[T: ClassTag](receiver: Receiver[T])：创建一个用户自定义的Receiver；
- socketTextStream：创建TCP socketReceiver，默认是utf-8的文本格式，以'\n'分隔；
- socketStream：创建TCP socketReceiver，用户提供自己的转化函数；
- rawSocketStream：创建SocketReceiver，相比上着，没有中间的解码转化所以比较高效；
- fileStream：创建监控HDFS目录的InputDStream，通过检测文件的修改时间来判断是否是新文件；
- binaryRecordsStream：创建二进制文件的监听InputDStream，使用了fileStream方法；
- queueStream：创建一个RDD队列流，底层调用了UnionRDD的方法将这些RDD转化为一个RDD，开启oneAtATime参数则每个RDD只取一个值，可以用于调试和测试；

在InputDStream的构造过程中，会将此输入流InputDStream添加到DStreamGraph的inputStreams数据结构中，
``` scala InputDStream.scala
ssc.graph.addInputStream(this)
```

还定义了一些DStream的其它方法：

- union： 多个DStreams合成一个DStream，底层调用了ssc.sc.union(rdds)；
- transform： 根据自定义的transformFunc生成新的DStream；
- addStreamingListener： 在listenerBus上增加一个StreamingListener对象，供JobScheduler的StreamingListenerBus对象监听输入流的ReceiverRateController对象；

还定义了一些控制启动与关闭的方法：

- start：启动StreamingContext。

StreamingContext的start方法启动过程中，会判断StreamingContext的状态，它有三个状态INITIALIZED、ACTIVE、STOP。只有状态为INITAILIZED才允许启动，主要有以下步骤：
1. 验证graph是否有效；
2. 设置Checkpoint；
3. 使用新的线程异步启动 JobScheduler ，启动后将状态由初始化状态INITIALIZED改为ACTIVE状态，JobScheduler的启动请见[JobGenerator 详解](https://github.com/lw-lin/CoolplaySpark/blob/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/2.2%20JobGenerator%20%E8%AF%A6%E8%A7%A3.md)；
5. 同时添加Streaming的shutdownHookRef，用于程序的异常终止，StreamingContext的shutdownHook优先级比SparkContext的值大1；
6. 往metricsSystem中注册streamingSource测量数据源；
7. 添加生成SparkUI中Streaming相关标签

- awaitTermination：等待Streaming程序的停止；
- stop：停止SparkStreaming程序，其中可以传入参数以表示是否同时停止相关的SparkContext，默认为true(这个参数在文件名流转化为数据流的时候应该设置为false，通过spark.streaming.stopSparkContextByDefault来设置);还有一个参数是是否优雅地停止(等待其它已经接收到的数据处理完毕再停止)；


### 引用

http://lqding.blog.51cto.com/9123978/1771017
https://github.com/lw-lin/CoolplaySpark/tree/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97
http://lqding.blog.51cto.com/9123978/1773912