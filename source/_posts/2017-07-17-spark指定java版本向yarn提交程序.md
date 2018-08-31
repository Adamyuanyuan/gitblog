---
title: spark在yarn中运行jdk8
toc: false
date: 2017-07-17 18:00:59
tags: spark开发
categories: spark开发
---

我们hadoop集群使用jdk版本为1.7，由于往5.X的ES中写数据必须要使用jdk1.8，该怎么办呢？
首先，把hadoop集群升级到jdk1.8是肯定可以的，但是这样代价太大。
我们通过如下两步操作，就可以在不升级集群的基础上，在yarn上运行用jdk1.8编译的spark程序。

1. yarn集群的每台nodeManager都需要安装jdk1.8，比如我们这边的安装路径是 `/usr/local/jdk1.8.0_111`
2. spark作业提交的时候，增加如下参数：
```
--conf "spark.yarn.appMasterEnv.JAVA_HOME=/usr/local/jdk1.8.0_111" --conf "spark.executorEnv.JAVA_HOME=/usr/local/jdk1.8.0_111"
```
这样就能使用jdk1.8啦，亲测可用