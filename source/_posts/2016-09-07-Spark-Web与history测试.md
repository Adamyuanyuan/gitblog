---
title: Spark Web与history web配置与测试
date: 2016-09-07 10:13:30
tags: spark
categories: spark
---

## Spark Web的查看

1. 运行任意一个yarn-client或者yarn-cluster模式的spark测试用例

``` bash
$ cd $SPARK_HOME
$ spark-submit --keytab testJars/op.keytab --principal op --master yarn-client --class SparkPi ./testJars/my.jar 4
```

2. 打开http://yarn-host:8088/cluster页面，找到正在运行的Spark测试用例

{% asset_img spark-yarn-web1.png %}

点击上图所示的AM，就进入了Spark的Web界面：下图就是Spark程序的web界面，值得注意的是，这个web界面会随着spark程序的运行结束而消失
{% asset_img spark-yarn-web2.png %}

## Spark history Web查看测试

在Kerberos环境下要启动spark history配置，需要在 spark -env下面开启如下配置 SPARK_HISTORY_OPTS：

``` bash spark-env.sh
# history需要的Kerberos配置
SPARK_HISTORY_OPTS="-Dspark.history.ui.port=8777 -Dspark.history.retainedApplications=10 -Dspark.history.fs.logDirectory=hdfs://ns/user/op/sparkHistoryServer -Dspark.history.kerberos.enabled=true -Dspark.history.kerberos.principal=op @HADOOP.CHINATELECOM.CN -Dspark.history.kerberos.keytab=/usr/op/sparkKerbersTest/spark-1.6.2-bin-hadoop2.6/conf/op.keytab"
```
然后通过 ./sbin/start-history-server.sh 命令启动history-server
然后登录 http://spark-client-ip:8777/ 即可查看 spark-history-web

{% asset_img spark-yarn-web3.png %}