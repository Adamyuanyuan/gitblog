---
title: Livy-server的搭建与简单测试
toc: true
date: 2017-02-24 17:18:30
tags:
- spark
- livy
categories: spark
---

半年前搭建并测试过livy，简单阅读过部分源码，现在因为openAPI项目，要深入了解livy-server了

## 搭建
搭建在官网中有，很简单，跳过 https://github.com/cloudera/livy#rest-api
启动的时候遇到的一个问题是会报如下错误：
```
java.io.IOException: Cannot write log directory /home/op/livy-server-0.3.0/logs
        at org.eclipse.jetty.util.RolloverFileOutputStream.setFile(RolloverFileOutputStream.java:219)
        at org.eclipse.jetty.util.RolloverFileOutputStream.<init>(RolloverFileOutputStream.java:166)
```
很简单，由于写日志的时候该目录不存在，所以只需要手动创建 logs目录即可

## 测试

### 启动session

我使用 Postman 来模拟它的rest api进行功能测试：
首先需要申请一个 session：

``` json
import json, pprint, requests, textwrap
host = 'http://someIp:8998'
data = {'kind': 'spark'}
headers = {'Content-Type': 'application/json'}
r = requests.post(host + '/sessions', data=json.dumps(data), headers=headers)
r.json()
```
需要注意的是，如果用postman来模拟的话，应该在Body中使用双引号的形式，单引号无法识别，比如：
```
{"code": "1+3"}
```
返回如下结果：
```
{
  "id": 2,
  "appId": null,
  "owner": null,
  "proxyUser": null,
  "state": "starting",
  "kind": "spark",
  "appInfo": {
    "driverLogUrl": null,
    "sparkUiUrl": null
  },
  "log": []
}

返回的header也很重要：

``` 
Content-Encoding → gzip
Content-Type → application/json
Date → Fri, 24 Feb 2017 09:17:07 GMT
Location → /sessions/2
Server → Jetty(9.2.16.v20160414)
Transfer-Encoding → chunked
```


进入对应机器，输入 `ps -ef | grep spark`，发现启动了三个相关的进程。
```
op        2002 63840  6 17:00 pts/9    00:00:37 /usr/java/jdk1.7.0_75/bin/java -cp /usr/lib/hadoop/lib/*:/usr/lib/spark/conf/:/usr/lib/spark/lib/spark-assembly-1.6.2-hadoop2.6.0-cdh5.4.7.jar:/usr/lib/spark/lib/datanucleus-rdbms-3.2.9.jar:/usr/lib/spark/lib/datanucleus-core-3.2.10.jar:/usr/lib/spark/lib/datanucleus-api-jdo-3.2.6.jar:/etc/hadoop/conf/ -Xms1g -Xmx1g -XX:MaxPermSize=256m org.apache.spark.deploy.SparkSubmit --properties-file /tmp/livyConf1238183037406660708.properties --class com.cloudera.livy.rsc.driver.RSCDriverBootstrapper spark-internal
op        2472 63840 10 17:05 pts/9    00:00:30 /usr/java/jdk1.7.0_75/bin/java -cp /usr/lib/hadoop/lib/*:/usr/lib/spark/conf/:/usr/lib/spark/lib/spark-assembly-1.6.2-hadoop2.6.0-cdh5.4.7.jar:/usr/lib/spark/lib/datanucleus-rdbms-3.2.9.jar:/usr/lib/spark/lib/datanucleus-core-3.2.10.jar:/usr/lib/spark/lib/datanucleus-api-jdo-3.2.6.jar:/etc/hadoop/conf/ -Xms1g -Xmx1g -XX:MaxPermSize=256m org.apache.spark.deploy.SparkSubmit --properties-file /tmp/livyConf8288202035580572985.properties --class com.cloudera.livy.rsc.driver.RSCDriverBootstrapper spark-internal
```
这表明**每提交一个livy的session，livy都会启动一个对应的进程，所以一定要考虑它的HA**

### 通过get方式查询状态
可以通过get方式查询这个session的状态
`http://10.142.78.39:8998/sessions/2`
返回是它的状态为空闲
```
{
  "id": 2,
  "appId": null,
  "owner": null,
  "proxyUser": null,
  "state": "idle",
  "kind": "spark",
  "appInfo": {
    "driverLogUrl": null,
    "sparkUiUrl": null
  },
  "log": []
}
```

### 提交Scala计算
session创建好了，就可以提交Scala计算了

```
statements_url = session_url + '/statements'
data = {'code': '1 + 1'}
r = requests.post(statements_url, data=json.dumps(data), headers=headers)
r.json()

```

提交后会马上返回：
```
{
  "id": 0,
  "state": "waiting",
  "output": null
}
```
waiting 说明正在计算。。。
header中返回了Location
```
Content-Encoding →gzip
Content-Type →application/json
Date →Fri, 24 Feb 2017 09:45:53 GMT
Location →/sessions/2/statements/0
Server →Jetty(9.2.16.v20160414)
Transfer-Encoding →chunked
```
获得location后，就可以通过location来获取结果啦：

```
statement_url = host + r.headers['location']
r = requests.get(statement_url, headers=headers)
```
结果如下：
```
{
  "id": 0,
  "state": "available",
  "output": {
    "status": "ok",
    "execution_count": 0,
    "data": {
      "text/plain": "res0: Int = 4"
    }
  }
}
```
结果是4

根据官网，跑个SparkPi吧：

```
{"code": "val NUM_SAMPLES = 100000;val count = sc.parallelize(1 to NUM_SAMPLES).map { i =>  val x = Math.random();  val y = Math.random();  if (x*x + y*y < 1) 1 else 0\n}.reduce(_ + _);println(\"Pi is roughly \" + 4.0 * count / NUM_SAMPLES)"}
```
获取后结果如下：

```
{
  "id": 3,
  "state": "available",
  "output": {
    "status": "ok",
    "execution_count": 3,
    "data": {
      "text/plain": "Pi is roughly 3.14464\nNUM_SAMPLES: Int = 100000\ncount: Int = 78616"
    }
  }
}
```
我故意改错了代码，会返回错误的结果：
```
{
  "id": 4,
  "state": "available",
  "output": {
    "status": "error",
    "execution_count": 4,
    "ename": "Error",
    "evalue": "<console>:1: error: integer number too large",
    "traceback": [
      "       val NUM_SAMPLES = 10000000000;val count = sc.parallelize(1 to NUM_SAMPLES).map { i =>  val x = Math.random();  val y = Math.random();  if (x*x + y*y < 1) 1 else 0\n",
      "                         ^"
    ]
  }
}
```
我故意往代码中加了一句睡眠`Thread.sleep(10000)`，当我马上请求结果的时候，会返回正在运行：
```
{
  "id": 6,
  "state": "running",
  "output": null
}
```

### 关闭session

好了，不玩了，关闭session吧：
```
session_url = 'http://localhost:8998/sessions/0'
requests.delete(session_url, headers=headers)
```
返回：
```
{
  "msg": "deleted"
}
```
如果session长时间不操作的话，会自动关闭，具体这个关闭策略是什么？待以后确认。

## 总结

这里只对livy的restAPI做了个简单的试验，具体的API见[livy官网](https://github.com/cloudera/livy#rest-api)
选择livy作为server端，是因为它相比于zeppelin以及jobserver功能比较专注，对yarn-cluster模式的支持使得它具有更好的HA以及可扩展性，接下来就是深入它，基于它完成spark能力的开放！
