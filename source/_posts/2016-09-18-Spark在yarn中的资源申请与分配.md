---
title: Spark在yarn中的资源申请与分配调研
date: 2016-09-18 11:09:21
toc: true
tags: [spark, yarn]
categories: spark
description: spark作业提交到yarn的时候，如果用户(wzfw)所在队列本来有500个executor的权限，但是他跑一个简单的程序根本不需要这么多的资源，只需要200个核就足够了，那他如果申请了500个核的话，是否需要全部分配给他？
---

本文解决遇到的以下问题：
*spark作业提交到yarn的时候，如果用户(wzfw)所在队列本来有500个executor的权限，但是他跑一个简单的程序根本不需要这么多的资源，只需要200个核就足够了，那他如果申请了400个核的话，是否需要全部分配给他？*

## 前言
目前我们所有的spark程序的分配都是靠参数设置固定的Executor数量进行资源预分配的，如果用户op在yarn的资源队列里可以申请到200个资源，那它就算跑占用资源很少的程序也能申请到200个核，这是不合理的

比如简单跑如下SparkPi程序，申请20个核：
``` bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-client --num-executors 20 lib/spark-examples-1.6.2-hadoop2.6.0.jar 
```
yarn中资源占用情况如下：
{% asset_img spark-yarn-allocation.png %}
可以看到，我就跑了一个SparkPi啊，竟然用了43G的内存，这样很不合理！

## 总述
Spark在yarn集群上运行的时候，一方面默认通过num-executors参数设置固定的Executor数量，每个application会独占所有预分配的资源直到整个生命周期的结束。Spark1.2后开始引入动态资源分配（Dynamic Resource Allocation）机制，支持资源弹性分配。

对于已知的业务负载，使用固定的集群资源配置是相对容易的；对于未知的业务负载，使用动态的集群资源分配方式可以满足负载的动态变化，这样集群的资源利用和业务负载的处理效率都会更加灵活。

动态资源分配测试在Spark1.2仅支持Yarn模式，从Spark1.6开始，支持standalone、Yarn、Mesos.这个特性默认是禁用的。
<!--more-->
## 动态资源分配的思想
简单来说，就是基于负载来动态调节Spark应用的资源占用，你的应用会在资源空闲的时候将其释放给集群，而后续用到的时候再重新申请。

### 动态资源分配策略
其实没有一个固定的方法可以预测一个executor后续是否马上会被分配去执行任务，或者一个新分配的执行器实际上是空闲的，所以我们需要一些试探性的方法，来决定是否申请或移除一个执行器。策略分为**请求策略**与**移除策略**：

#### 请求策略

开启动态分配策略后，application会在task因没有足够资源被挂起的时候去动态申请资源，这种情况意味着该application现有的executor无法满足所有task并行运行。spark一轮一轮的申请资源，当有task挂起或等待spark.dynamicAllocation.schedulerBacklogTimeout(默认1s)时间的时候，会开始动态资源分配；之后会每隔spark.dynamicAllocation.sustainedSchedulerBacklogTimeout(默认1s)时间申请一次，直到申请到足够的资源。**每次申请的资源量是指数增长的，即1,2,4,8等**。
之所以采用指数增长，出于两方面考虑：其一，开始申请的少是考虑到可能application会马上得到满足；其次要成倍增加，是为了如果application需要很多资源，而该方式可以在很少次数的申请之后得到满足。
（这段指数增长的策略可以根据实际情况通过修改源码来修改）

#### 资源回收策略
当application的executor空闲时间超过spark.dynamicAllocation.executorIdleTimeout（默认60s）后，就会被回收。

## 配置思路
### 启动 external shuffle service
要使用这一特性有两个前提条件。首先，你的应用必须设置 spark.dynamicAllocation.enabled 为 true。其次，你必须在每个节点上启动一个外部混洗服务（external shuffle service），并在你的应用中将 spark.shuffle.service.enabled 设为true。外部混洗服务的目的就是为了在删除执行器的时候，能够保留其输出的混洗文件（本文后续有更详细的描述）。启用外部混洗的方式在各个集群管理器上各不相同：

在Spark独立部署的集群中，你只需要在worker启动前设置 spark.shuffle.server.enabled 为true即可。

在YARN模式下，混洗服务需要按以下步骤在各个NodeManager上启动：

1. 首先按照YARN profile 构建Spark。如果你已经有打好包的Spark，可以忽略这一步。
2. 找到 spark-<version>-yarn-shuffle.jar。如果你是自定义编译，其位置应该在 ${SPARK_HOME}/network/yarn/target/scala-<version>，否则应该可以在 lib 目录下找到这个jar包。
3. 将该jar包添加到NodeManager的classpath路径中。
4. 配置各个节点上的yarn-site.xml，将 spark_shuffle 添加到 yarn.nodemanager.aux-services 中，然后将 yarn.nodemanager.aux-services.spark_shuffle.class 设为 org.apache.spark.network.yarn.YarnShuffleService，并将 spark.shuffle.service.enabled 设为 true。
5. 最后重启各节点上的NodeManager。

所有相关的配置都是可选的，并且都在 spark.dynamicAllocation.* 和 spark.shuffle.service.* 命名空间下。更详细请参考：[configurations page](http://spark.apache.org/docs/latest/configuration.html#dynamic-allocation)。

### 外部混洗服务external shuffle service
非动态分配模式下，执行器可能的退出原因有执行失败或者相关Spark应用已经退出。不管是哪种原因，执行器的所有状态都已经不再需要，可以丢弃掉。但在动态分配的情形下，执行器有可能在Spark应用运行期间被移除。这时候，如果Spark应用尝试去访问该执行器存储的状态，就必须重算这一部分数据。因此，Spark需要一种机制，能够优雅地关闭执行器，同时还保留其状态数据。

这种需求对于混洗操作尤其重要。混洗过程中，Spark执行器首先将map输出写到本地磁盘，同时执行器本身又是一个文件服务器，这样其他执行器就能够通过该执行器获得对应的map结果数据。一旦有某些任务执行时间过长，动态分配有可能在混洗结束前移除任务异常的执行器，而这些被移除的执行器对应的数据将会被重新计算，但这些重算其实是不必要的。

要解决这一问题，就需要用到一个外部混洗服务（external shuffle service），该服务在Spark 1.2引入。该服务在每个节点上都会启动一个不依赖于任何Spark应用或执行器的独立进程。一旦该服务启用，Spark执行器不再从各个执行器上获取shuffle文件，转而从这个service获取。这意味着，任何执行器输出的混洗状态数据都可能存留时间比对应的执行器进程还长。

除了混洗文件之外，执行器也会在磁盘或者内存中缓存数。一旦执行器被移除，其缓存数据将无法访问。这个问题目前还没有解决。或许在未来的版本中，可能会采用外部混洗服务类似的方法，将缓存数据保存在堆外存储中以解决这一问题。


## 配置说明
配置文件：
$SPARK_HOME/conf/spark-defaults.conf
$HADOOP_HOME/conf/yarn-site.xml

### Spark配置说明
在spark-defaults.conf 中添加
``` bash spark-defaults.conf
spark.shuffle.service.enabled true   # 开启外部shuffle服务，开启这个服务可以保护executor的shuffle文件，安全移除executor，在Yarn模式下这个shuffle服务以org.apache.spark.yarn.network.YarnShuffleService实现
spark.shuffle.service.port 7337 # Shuffle Service服务端口，必须和yarn-site中的一致
spark.dynamicAllocation.enabled true  # 开启动态资源分配
spark.dynamicAllocation.minExecutors 1  # 每个Application最小分配的executor数
spark.dynamicAllocation.maxExecutors 30  # 每个Application最大并发分配的executor数
spark.dynamicAllocation.schedulerBacklogTimeout 1s # 任务待时间（超时便申请新资源)默认60秒
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout 5s #  再次请求等待时间，默认60秒
spark.dynamicAllocation.executorIdleTimeout # executor闲置时间（超过释放资源）默认600秒
```
### yarn的配置
#### 添加相应的jar包spark-<version>-yarn-shuffle.jar
如果是自己编译的spark，可以在$SPARK_HOME/network/yarn/target/scala-<version>下面找到
是预编译的，直接在$SPARK_HOME/lib/下面找到
找到jar包后，将其添加到每个nodemanager的classpath下面(或者直接放到yarn的lib目录中,${HADOOP_HOME}/share/hadoop/yarn/lib/)

#### 配置yarn-site.xml文件
在所有节点的yarn-site.xml中，为yarn.nodemanager.aux-services配置项新增spark_shuffle这个值（注意是新增，在原有value的基础上逗号分隔新增即可）
``` xml yarn-site.xml
<property>
  <name>spark.shuffle.service.enabled</name>
  <value>true</value>
</property>
<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle,spark_shuffle</value>
</property>
<property>
  <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
  <value>org.apache.spark.network.yarn.YarnShuffleService</value>
</property>
```
#### 重启所有的节点

### 注意
当开启了动态资源分配（spark.dynamicAllocation.enabled），num-executor选项将不再兼容，如果设置了num-executor，那么动态资源分配将被关闭


## 引用
[spark1.6.2作业调度官网](http://spark.apache.org/docs/1.6.2/job-scheduling.html#dynamic-resource-allocation)
[spark1.6.2作业调度翻译版](http://ifeve.com/spark-schedule/)
[Apache Spark Resource Management and YARN App Models](http://blog.cloudera.com/blog/2014/05/apache-spark-resource-management-and-yarn-app-models/)
[jira/browse/YARN-1197--Support changing resources of an allocated container](https://issues.apache.org/jira/browse/YARN-1197)
[Spark集群资源动态分配](http://hejunhao.me/archives/675)
[spark动态资源分配在yarn（hadoop）的配置](http://blog.sina.com.cn/s/blog_a29dec8d0102vfwx.html)
[Spark Executors在YARN上的动态分配](https://www.linkedin.com/pulse/spark-executors%E5%9C%A8yarn%E4%B8%8A%E7%9A%84%E5%8A%A8%E6%80%81%E5%88%86%E9%85%8D-victor-wang)