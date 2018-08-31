---
title: Apache-Eagle定义一个Application
toc: true
date: 2017-03-10 18:18:23
tags: 
- spark
- eagle 
categories: eagle
---

为了监控大数据平台的大量组件与应用，我们决定引入 Apache Eagle()，Apache Eagle是由Ebay贡献并在2017年初成为了顶级项目。。。它的核心就是用一个实时计算平台（Storm），接收数据（kafka等），然后处理，然后存hbase，然后报警。由于缺乏文档，所以最简单的配置使用也是踩了不少坑：
当前版本：Eagle 0.5.0 Spark 1.6.2


## 配置使用

http://10.142.78.74:8090/#/integration/applicationList

http://10.142.78.100:8080/topology.html?id=SPARK_HISTORY_JOB_APP_TESTENV-38-1489136484

http://10.142.78.100:60010/master-status

http://www.yiibai.com/hbase/


## 设计
### alert engine
#### 高层次设计

从高层来看，alert engine 是一个元数据驱动的storm拓扑，它包括了多个模块协同工作：

- Admin Service - 提供了元数据管理和拓扑管理的API。其中：Metadata store 是 admin service API的具体实现.
- Alert Engine Topology on Storm ：通用的Storm 拓扑
- Coordinator 协调器： alert engine拓扑的调度程序。它是一个后端调度程序，用于新策略的加载，资源分配，并且暴露了一些内部的API用于管理；
- Zookeeper：作为通信和警报引擎之间的通信。

{% asset_img eagle_1.png %}

## 原理
Spark History Job Monitor主要分为两大步骤，在代码中的表现为一个 SparkHistoryJobSpout 和一个 SparkHistoryJobParseBolt：

SparkHistoryJobSpout：从 rm 提供的metric中，解析出 已经完成的spark job：COMPLETE_SPARK_JOB，得到 applicationID，发给下一个Bolt
SparkHistoryJobParseBolt：接收到上游发过来的appId后，通过spark的规则，将其

## 配置 Spark History Job Monitor

分别点击 `Integration` -> `Sites` -> `Edit` 进入应用配置界面；
找到 `Spark History Job Monitor` 配置，点击右边的 `编辑`按钮，即可编辑
对于 Spark History Job Monitor，由于我们集群是Kerberos的，需要配置以下参数，

#### Environment
##### Execution Mode  选择 Cluster Mode

##### Execution File  
默认为：  /usr/local/eagle-0.5.0-SNAPSHOT/lib/eagle-topology-0.5.0-SNAPSHOT-assembly.jar 
可以不用改

#### General
##### resource manager url  指的是yarn的那个界面
	http://10.142.78.36:8090/
##### hdfs url 指的是 hdfs的路径 
	hdfs://10.142.78.98:54310/
##### hdfs base path for spark job data  指的是spark history server配置的日志写在hdfs中的路径
	hdfs:///user/op/sparkHistoryServe

#### Advanced
这里主要配的是一些storm以及spark的一些相关参数，可以先不用配

#### Custom 如果hdfs是Kerberos的话，需要配置Kerberos相关参数
需要增加如下：

``` bash
dataSourceConfig.hdfs.keytab.file hdfs@HADOOP.CHINATELECOM.CN
dataSourceConfig.hdfs.kerberos.principal /etc/hadoop/conf/hdfs.keytab
```

#### custom
``` json
{
    "workers": "3",
    "topology.numOfSpoutExecutors": "1",
    "topology.numOfSpoutTasks": "1",
    "topology.numOfParseBoltExecutors": "6",
    "topology.numOfParserBoltTasks": "6",
    "topology.spoutCrawlInterval": "60000",
    "topology.requestLimit": "100",
    "topology.message.timeout.secs": "600",
    "service.flushLimit": "500",
    "dataSourceConfig.rm.url": "",
    "dataSourceConfig.hdfs.fs.defaultFS": "hdfs://xxxxx",
    "dataSourceConfig.hdfs.baseDir": "/logs/spark-events",
    "spark.jobConf.additional.info": "",
    "spark.defaultVal.spark.executor.memory": "1g",
    "spark.defaultVal.spark.driver.memory": "1g",
    "spark.defaultVal.spark.driver.cores": "1",
    "spark.defaultVal.spark.executor.cores": "1",
    "spark.defaultVal.spark.yarn.am.memory": "512m",
    "spark.defaultVal.spark.yarn.am.cores": "1",
    "spark.defaultVal.spark.yarn.executor.memoryOverhead.factor": "10",
    "spark.defaultVal.spark.yarn.driver.memoryOverhead.factor": "10",
    "spark.defaultVal.spark.yarn.am.memoryOverhead.factor": "10",
    "spark.defaultVal.spark.yarn.overhead.min": "384m",
    "dataSourceConfig.hdfs.dfs.namenode.rpc-address.xxxxx.nn1": "yyyyy",
    "dataSourceConfig.hdfs.dfs.namenode.rpc-address.xxxxx.nn2": "zzzzz",
    "dataSourceConfig.hdfs.dfs.client.failover.proxy.provider.xxxxx": "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider",
    "dataSourceConfig.hdfs.dfs.nameservices": "xxxxx",
    "dataSourceConfig.hdfs.dfs.ha.namenodes.xxxxx": "nn1,nn2",
    "dataSourceConfig.hdfs.hdfs.kerberos.principal": "aaaaaaa",
    "dataSourceConfig.hdfs.dfs.data.transfer.saslproperties.resolver.class": "org.apache.hadoop.security.WhitelistBasedResolver",
    "dataSourceConfig.hdfs.hdfs.keytab.file": "/home/storm/.keytab/b_eagle.keytab",
    "dataSourceConfig.hdfs.dfs.data.transfer.protection": "authentication,privacy",
    "dataSourceConfig.hdfs.dfs.encrypt.data.transfer.cipher.suites": "AES/CTR/NoPadding"
}
```

``` json 我们的
{
	"workers": "1",
	"topology.numOfSpoutExecutors": "1",
	"topology.numOfSpoutTasks": "4",
	"topology.numOfParseBoltExecutors": "1",
	"topology.numOfParserBoltTasks": "4",
	"topology.spoutCrawlInterval": "10000",
	"topology.message.timeout.secs": "300",
	"service.flushLimit": "500",
	"dataSourceConfig.rm.url": "http://10.142.78.36:8090/",
	"dataSourceConfig.hdfs.fs.defaultFS": "hdfs://ns",
	"dataSourceConfig.hdfs.baseDir": "/user/op/sparkHistoryServer",
	"spark.jobConf.additional.info": "",
	"spark.defaultVal.spark.executor.memory": "1g",
	"spark.defaultVal.spark.driver.memory": "1g",
	"spark.defaultVal.spark.driver.cores": "1",
	"spark.defaultVal.spark.executor.cores": "1",
	"spark.defaultVal.spark.yarn.am.memory": "512m",
	"spark.defaultVal.spark.yarn.am.cores": "1",
	"spark.defaultVal.spark.yarn.executor.memoryOverhead.factor": "10",
	"spark.defaultVal.spark.yarn.driver.memoryOverhead.factor": "10",
	"spark.defaultVal.spark.yarn.am.memoryOverhead.factor": "10",
	"spark.defaultVal.spark.yarn.overhead.min": "384m",

	"dataSourceConfig.hdfs.dfs.namenode.kerberos.principal": "hdfs/_HOST@HADOOP.CHINATELECOM.CN",
	"dataSourceConfig.hdfs.hadoop.security.authorization": "true",
	"dataSourceConfig.hdfs.dfs.data.transfer.protection": "authentication,privacy",
	"dataSourceConfig.hdfs.dfs.namenode.rpc-address.ns.nn1": "NM-ITC-NF8460M3-303-011:54310",
	"dataSourceConfig.hdfs.dfs.namenode.rpc-address.ns.nn2": "NM-ITC-NF8460M3-303-012:54310",
	"dataSourceConfig.hdfs.dfs.nameservices": "ns",
	"dataSourceConfig.hdfs.dfs.client.failover.proxy.provider.ns": "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider",
	"dataSourceConfig.hdfs.java.security.krb5.kdc": "test-bdd-073",
	"dataSourceConfig.hdfs.dfs.ha.namenodes.ns": "nn1,nn2",
	"dataSourceConfig.hdfs.dfs.encrypt.data.transfer.cipher.suites": "AES/CTR/NoPadding",
	"dataSourceConfig.hdfs.hdfs.kerberos.principal": "hdfs@HADOOP.CHINATELECOM.CN",
	"dataSourceConfig.hdfs.dfs.data.transfer.saslproperties.resolver.class": "org.apache.hadoop.security.WhitelistBasedResolver",
	"dataSourceConfig.hdfs.hadoop.security.authentication": "kerberos",
	"dataSourceConfig.hdfs.java.security.krb5.conf": "/etc/krb5.conf",
	"dataSourceConfig.hdfs.java.security.krb5.realm": "HADOOP.CHINATELECOM.CN",
	"dataSourceConfig.hdfs.hdfs.keytab.file": "/tmp/hdfs.keytab"
}
```
``` json ,我们的 MR history job
{
	"workers": "2",
	"stormConfig.mrHistoryJobSpoutTasks": "4",
	"stormConfig.jobKafkaSinkTasks": "1",
	"stormConfig.taskAttemptKafkaSinkTasks": "1",
	"endpointConfig.hdfs.fs.defaultFS": "hdfs://ns",
	"endpointConfig.basePath": "/history-yarn/done",
	"endpointConfig.mrHistoryServerUrl": "http://10.142.78.40:19890",
	"endpointConfig.timeZone": "Etc/GMT-8",
	"dataSinkConfig.MAP_REDUCE_JOB_STREAM.topic": "map_reduce_job_testenv",
	"dataSinkConfig.MAP_REDUCE_TASK_ATTEMPT_STREAM.topic": "map_reduce_task_attempt_testenv",
	"dataSinkConfig.brokerList": "10.142.78.100:9092",
	"dataSourceConfig.zkConnection": "10.142.78.98:2181,10.142.78.99:2181,10.142.78.100:2181",
	"dataSinkConfig.serializerClass": "kafka.serializer.StringEncoder",
	"dataSinkConfig.keySerializerClass": "kafka.serializer.StringEncoder",
	"dataSinkConfig.producerType": "async",
	"dataSinkConfig.numBatchMessages": "4096",
	"dataSinkConfig.maxQueueBufferMs": "5000",
	"dataSinkConfig.requestRequiredAcks": "0",
	
	"endpointConfig.hdfs.dfs.namenode.kerberos.principal": "hdfs/_HOST@HADOOP.CHINATELECOM.CN",
	"endpointConfig.hdfs.hadoop.security.authorization": "true",
	"endpointConfig.hdfs.dfs.data.transfer.protection": "authentication,privacy",
	"endpointConfig.hdfs.dfs.namenode.rpc-address.ns.nn1": "NM-ITC-NF8460M3-303-011:54310",
	"endpointConfig.hdfs.dfs.namenode.rpc-address.ns.nn2": "NM-ITC-NF8460M3-303-012:54310",
	"endpointConfig.hdfs.dfs.nameservices": "ns",
	"endpointConfig.hdfs.dfs.client.failover.proxy.provider.ns": "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider",
	"endpointConfig.hdfs.java.security.krb5.kdc": "test-bdd-073",
	"endpointConfig.hdfs.dfs.ha.namenodes.ns": "nn1,nn2",
	"endpointConfig.hdfs.dfs.encrypt.data.transfer.cipher.suites": "AES/CTR/NoPadding",
	"endpointConfig.hdfs.hdfs.kerberos.principal": "hdfs@HADOOP.CHINATELECOM.CN",
	"endpointConfig.hdfs.dfs.data.transfer.saslproperties.resolver.class": "org.apache.hadoop.security.WhitelistBasedResolver",
	"endpointConfig.hdfs.hadoop.security.authentication": "kerberos",
	"endpointConfig.hdfs.java.security.krb5.conf": "/etc/krb5.conf",
	"endpointConfig.hdfs.java.security.krb5.realm": "HADOOP.CHINATELECOM.CN",
	"endpointConfig.hdfs.hdfs.keytab.file": "/tmp/hdfs.keytab"
}
```


#### 引用
https://eagle.apache.org/
http://www.csdn.net/article/2015-10-29/2826076