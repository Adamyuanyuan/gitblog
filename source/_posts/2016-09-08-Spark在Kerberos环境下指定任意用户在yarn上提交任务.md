---
title: Spark在Kerberos环境下指定任意用户在yarn上提交任务
date: 2016-09-08 11:10:59
tags: 
- spark
- kerberos
categories:
- spark
---

众所周知，Spark在Kerberos环境下提交任务有两种方式，分别是先kinit的方式和通过 --keytab的方式：

``` bash
[op]$ spark-submit --keytab testJars/op.keytab --principal op --master local --class SparkPi ./testJars/my.jar 4
```


Spark在Kerberos环境下可以在提交任务时通过指定用户的keytab和principal来提交任务，比如：

``` bash
# 事先进行kinit的方式
[op]$ kinit -kt op.keytab op
[op]$ spark-submit --master local --class SparkPi ./testJars/my.jar 4

# 提交keytab的方式
[op]$ spark-submit --keytab testJars/op.keytab --principal op --master local --class SparkPi ./testJars/my.jar 4
```

其实还可以模拟其它用户的方式提交任务，比如使用ts账户提交：

``` bash
[op]$ spark-submit --keytab testJars/ts.keytab --principal ts@HADOOP.CHINATELECOM.CN --master local --class SparkPi ./testJars/my.jar 4
```

当然没有那么简单，如果想要使用ts账户执行程序，需要进行如下设置：

#### 模拟其它用户需要的条件

1. ts要在KDC下生成对应的keytab和principal；
2. 要在hadoop集群的所有机器上创建ts账户：

``` bash
sudo groupadd ts
sudo useradd -g ts ts
```

值得注意的是，如果要在yarn中模拟其它用户执行，需要在集群中所有机器上增加该用户。

后期有时间了详细说明原因。