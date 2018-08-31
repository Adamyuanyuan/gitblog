---
title: flume1.7使用KafkaSource采集大量数据
toc: true
date: 2017-03-23 11:01:43
tags: flume
categories: flume
---

我们需要把spark Streaming的大量数据写入HDFS或者ES，为了增加稳定性，先将数据写入 kafka集群，然后使用 flume导入HDFS，数据量平均为 20W/s;

由于kafka版本为0.10，为了提高效率，我们使用了1.7版本的flume，它是直接消费kafka集群的，有以下坑：

#### 配置方法
先说说高效的配置方法吧：
同时参照[flume官网文档](https://flume.apache.org/FlumeUserGuide.html#kafka-source)和[kafka官网文档](http://kafka.apache.org/documentation/#consumerapi)。由于新出来，所以官网是最好的，并且因为kafka的consumer配置非常多，在flume官网中默认的配置很少，所以还是需要根据具体情况配置一些好用的参数。

#### Commit cannot be completed due to group rebalance

```
2017-03-23 10:32:32,509 (PollableSourceRunner-KafkaSource-r1) [ERROR - org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:550)] Error ILLEGAL_GENERATION occurred while committing offsets for group flume_test
2017-03-23 10:32:32,509 (PollableSourceRunner-KafkaSource-r1) [ERROR - org.apache.flume.source.kafka.KafkaSource.doProcess(KafkaSource.java:314)] KafkaSource EXCEPTION, {}
org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed due to group rebalance
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:552)
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:493)
        at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:665)
        at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:644)
        at org.apache.kafka.clients.consumer.internals.RequestFuture$1.onSuccess(RequestFuture.java:167)
        at org.apache.kafka.clients.consumer.internals.RequestFuture.fireSuccess(RequestFuture.java:133)
        at org.apache.kafka.clients.consumer.internals.RequestFuture.complete(RequestFuture.java:107)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler.onComplete(ConsumerNetworkClient.java:380)
        at org.apache.kafka.clients.NetworkClient.poll(NetworkClient.java:274)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.clientPoll(ConsumerNetworkClient.java:320)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:213)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:193)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:163)
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.commitOffsetsSync(ConsumerCoordinator.java:358)
        at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:968)
        at org.apache.flume.source.kafka.KafkaSource.doProcess(KafkaSource.java:304)
        at org.apache.flume.source.AbstractPollableSource.process(AbstractPollableSource.java:60)
        at org.apache.flume.source.PollableSourceRunner$PollingRunner.run(PollableSourceRunner.java:133)
        at java.lang.Thread.run(Thread.java:745)
```

原因是我的数据量特别大，导致每次消费consumer进行poll的时候耗时太久，导致发送hearbeat间隔太长，coordinator认为consumer死了，就发生了rebalance；

解决方法：
增大参数：heartbeat.interval.ms - This tells Kafka wait the specified amount of milliseconds before it consider the consumer will be considered "dead"

缩小参数：max.partition.fetch.bytes - This will limit the amount of messages (up to) the consumer will receive when polling. 

实际解决的时候，我并没有缩小 max.partition.fetch.bytes 参数，因为我觉得一次多拉点也好


## 引用
http://blog.csdn.net/xianzhen376/article/details/51802736
https://github.com/ajkret/kafka-sample
http://stackoverflow.com/questions/35658171/kafka-commitfailedexception-consumer-exception
http://kaimingwan.com/post/kafka/kafkawen-ti-shou-ji