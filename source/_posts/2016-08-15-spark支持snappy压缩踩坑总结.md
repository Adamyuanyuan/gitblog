---
title: spark支持snappy压缩踩坑总结
date: 2016-08-15 10:16:54
tags: spark
categories: spark
---

### 配置snappy压缩

首先在/usr/lib/hadoop/lib/目录下配置lzo相关的包，
然后在spark客户端配置如下：

``` bash spark-default.conf
spark.driver.extraClassPath /usr/lib/hadoop/lib/*
spark.driver.extraLibraryPath /usr/lib/hadoop/lib/native
spark.executor.extraClassPath /usr/lib/hadoop/lib/*
spark.executor.extraLibraryPath /usr/lib/hadoop/lib/native
```
如上配置，即可，但是为了得到这么小小的一点配置，浪费了三天的时间啊，网上的资料都是转载，无法解决问题。最新的官网的配置文件中并没有关于spark.executor.extraClassPath的配置，查了源码才得知，作为教训。以后出现问题要冷静思考，不要简单的去网上搜索，先判断问题出现的原因，知其所以然，必要时要去源码中查询，否则会浪费很多时间，走很多弯路。

### 踩坑集锦

首先，会遇到这个错误：

``` bash
Compression codec com.hadoop.compression.lzo.LzoCodec not found
```

原因是spark-env.sh的配置文件缺少关联hadoop的配置语句

``` bash
export SPARK_LIBRARY_PATH=$SPARK_LIBRARY_PATH:/usr/lib/hadoop/lib/native/:/usr/lib/hadoop/lib/*
```

然后yarn-cluster模式下snappy压缩总会报错：

``` java
16/08/08 19:05:03 DEBUG util.NativeCodeLoader: java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
16/08/08 19:05:03 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
1）
16/08/08 19:05:03 DEBUG util.PerformanceAdvisory: Both short-circuit local reads and UNIX domain socket are disabled.
16/08/08 19:05:03 DEBUG sasl.DataTransferSaslUtil: DataTransferProtocol not using SaslPropertiesResolver, no QOP found in configuration for dfs.data.transfer.protection
16/08/08 19:05:03 ERROR lzo.GPLNativeCodeLoader: Could not load native gpl library
java.lang.UnsatisfiedLinkError: no gplcompression in java.library.path
 at java.lang.ClassLoader.loadLibrary(ClassLoader.java:1886)
 at java.lang.Runtime.loadLibrary0(Runtime.java:849)
 at java.lang.System.loadLibrary(System.java:1088)
 at com.hadoop.compression.lzo.GPLNativeCodeLoader.<clinit>(GPLNativeCodeLoader.java:32)
 at com.hadoop.compression.lzo.LzoCodec.<clinit>(LzoCodec.java:71)
 at java.lang.Class.forName0(Native Method)
 at java.lang.Class.forName(Class.java:274)
 at org.apache.hadoop.conf.Configuration.getClassByNameOrNull(Configuration.java:2013)
 at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:1978)
 at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:128)
 at org.apache.hadoop.io.compress.CompressionCodecFactory.<init>(CompressionCodecFactory.java:175)
 at org.apache.hadoop.mapred.TextInputFormat.configure(TextInputFormat.java:45)
 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```

这个究其原因就是程序运行的那个节点找不到lzo解压的包导致的，官网中只说明了 spark.driver.extraClassPath，但并没有说明配置spark.executor.extraClassPath 与 spark.executor.extraLibraryPath，导致不管怎么根据网上博客或者官网配置配，executor还是找不到lzo压缩相关的包，后来聪哥通过源码查看才发现有这么一个参数配置，只是各类文档中都没有，加上就ok了~
