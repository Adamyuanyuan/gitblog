---
title: Spark在Kerberos下连接使用Hbase的配置
toc: false
date: 2016-09-18 17:06:06
tags: [spark, hbase, kerberos]
categories: spark
---

复制HBase目录下的lib文件到spark目录/lib/hbase。spark依赖此lib，但直接指定到Hbase下的lib目录的话又会出错
清单如下：guava-12.0.1.jar htrace-core-3.1.0-incubating.jar protobuf-java-2.5.0.jar   这三个jar加上以hbase开头所有jar，其它就不必了，全部复制会引起报错。
``` bash 
cd $SPARK_HOME/lib/hbase
cp /usr/lib/hbase/lib/hbase-* ./
cp /usr/lib/hbase/lib/guava-12.0.1.jar ./
cp /usr/lib/hbase/lib/htrace-core-3.1.0-incubating.jar ./
cp /usr/lib/hbase/lib/protobuf-java-2.5.0.jar ./
```
然后在spark客户端配置如下：
也就是增加到classpath
``` bash spark-default.conf
spark.driver.extraClassPath /usr/lib/hadoop/lib/*:/usr/op/sparkKerbersTest/spark-1.6.2-bin-hadoop2.6/lib/hbase/*
```
就可以了