---
title: spark中读取hdfs文件简记
toc: true
date: 2016-12-19 10:46:18
tags:
- spark开发
categories: spark开发
---

使用spark的API读取hdfs的方法是：
``` scala
val lines: RDD[String] = sc.textFile(filePath)
```
如果该文件不存在，就会报错，报错多次，就会奔溃

``` bash
16/12/16 16:53:12 ERROR JobScheduler: Error running job streaming job 1481878392000 ms.0
org.apache.hadoop.mapred.InvalidInputException: Input path does not exist: hdfs://ns/data/.../4811_000.txt
	at org.apache.hadoop.mapred.FileInputFormat.singleThreadedListStatus(FileInputFormat.java:285)
	at org.apache.hadoop.mapred.FileInputFormat.listStatus(FileInputFormat.java:228)
	at org.apache.hadoop.mapred.FileInputFormat.getSplits(FileInputFormat.java:313)
	at org.apache.spark.rdd.HadoopRDD.getPartitions(HadoopRDD.scala:199)
	at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:239)
	at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:237)
	at scala.Option.getOrElse(Option.scala:120)

	...

16/12/16 16:53:12 ERROR ApplicationMaster: User class threw exception: org.apache.hadoop.mapred.InvalidInputException: Input path does not exist: hdfs://ns/data...00.txt
org.apache.hadoop.mapred.InvalidInputException: Input path does not exist: hdfs://ns/data/hjpt/...000.txt
	at org.apache.hadoop.mapred.FileInputFormat.singleThreadedListStatus(FileInputFormat.java:285)
	at org.apache.hadoop.mapred.FileInputFormat.listStatus(FileInputFormat.java:228)
	at org.apache.hadoop.mapred.FileInputFormat.getSplits(FileInputFormat.java:313)
	at org.apache.spark.rdd.HadoopRDD.getPartitions(HadoopRDD.scala:199)
	at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:239)
	at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:237)
	at scala.Option.getOrElse(Option.scala:120)
	at org.apache.spark.rdd.RDD.partitions(RDD.scala:237)
```

由于spark中hdfs读是lazy的，所以无法使用try-catch把它装住，即使用try-catch将其包住也会报错。
所以目前使用的解决方法是需要在读取该文件之前检验该文件是否存在。
在spark中的实现不需要再次指定hadoopConf，只要从sc中拿就可以了：

``` scala
val conf = sc.hadoopConfiguration
val fs = org.apache.hadoop.fs.FileSystem.get(conf)
val exists = fs.exists(new org.apache.hadoop.fs.Path("/path/on/hdfs/to/SUCCESS.txt"))
```

实际实现如下：
``` scala
...
val sc: SparkContext = eachRdd.sparkContext
val hadoopConf: Configuration = sc.hadoopConfiguration
val fs: FileSystem = org.apache.hadoop.fs.FileSystem.get(hadoopConf)

// 这里是否不需要collect？
val lines: Array[(String, FtpMap)] = oiddRdd.collect()
// 文件名流转化为文件数据流
lines.foreach {
eachFileJson: (String, FtpMap) => {
  val topic: String = eachFileJson._1
  printLog.info("topic: " + topic)
  val fileJson = eachFileJson._2
  val filePath = fileJson.file_path
  val fileExists: Boolean = try {
    fs.exists(new org.apache.hadoop.fs.Path(filePath))
  } catch {
    case e: Exception => {
      printLog.error("Exception: filePath:" + filePath + " e:" + e)
      false
    }
  }
  if (fileExists) {
    val lines: RDD[String] = sc.textFile(filePath)
    ...
  }
}
```

值得注意的是：

如果读取一个hdfs目录下的所有文件，当文件的数目非常多，比如说有一亿个文件，由于spark读hdfs文件是lazy的，它在读取hdfs下该文件的列表的时候，会先将这个列表保存到内存中，但是如果在这期间hdfs文件被删除了，则还是会发生文件不存在的错误，所以以后遇到这类问题的时候要注意。