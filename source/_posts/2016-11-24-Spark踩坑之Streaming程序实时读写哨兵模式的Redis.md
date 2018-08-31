---
title: Spark踩坑之Streaming程序实时读写哨兵模式的Redis
toc: true
date: 2016-11-24 10:51:50
tags: 
- spark streaming
- redis
- spark开发
categories: spark开发
---

## 背景
Spark Streaming程序使用Redis保存出现在景区中的用户，用以识别用户是否是第一次进入景区，要分别进行读操作和写操作，为了程序的高可用性，Redis我们使用的是哨兵模式（满满的，都是坑）；

Redis哨兵模式，顾名思义，就是Redis有三台机器作为一主两备，然后有三个哨兵（三个端口），告诉你Master是哪一台，然后你再去访问master，这样的话，一台机器由于负载发生了主备切换，对我们来说，对外的端口是统一的（哨兵）；

我们Redis的版本是3.2.4，连接Redis，我们的程序使用了开源的jedis作为redis的连接器，jedis的版本是2.8.1，关于jedis的使用，我参考的是jedis源码的单元测试程序，这个可以在github中找到。

## 踩坑历程

### hgetAll不能读取太大量的数据
在设计程序的时候，我们使用了hash作为数据的存储结构，该hash可以存储40亿条记录，一开始，为了减少对Redis的压力，我们选择的策略是对于每个batch，我们都统一读取全部的数据，然后在内存中筛选计算的方法，可以极大减少对redis的访问；

``` scala
val sentinelPool = InternalRedisClient.getSentinelPool
var phoneMap: util.Map[String, String] = new util.HashMap[String, String]()
// printLog.debug( "sentinelPool NumIdle0: " + sentinelPool.getNumIdle + " Active0: " + sentinelPool.getNumActive)
var jedis1: Jedis = null
try {
  jedis1 = sentinelPool.getResource
  // printLog.debug( "sentinelPool NumIdle1: " + sentinelPool.getNumIdle + " Active1: " + sentinelPool.getNumActive)
  // 这里当redis中该HSet数据量很大的时候（五十万条），一次读取需要很长时间，对性能影响很大，
  // 故当数据条数大于一定值的时候，不用此方法
  phoneMap = jedis1.hgetAll(redisHashKey)
  // printLog.debug( "sentinelPool NumIdle3: " + sentinelPool.getNumIdle + " Active3: " + sentinelPool.getNumActive)
  printLog.debug("phoneSet_1: " + phoneMap)
} finally {
  if (jedis1 != null) {
    printLog.debug("close jedis1")
    jedis1.close()
  }
}
```

```scala

但是当程序运行一段时间后，发现延迟很高，我们也没有想到数据量会有这么大，发现当数据量超过10W条这个级别后，每次读取全部数据的耗时将会很大，而且对redis的压力反而变大了，所以我们弃用了一次读取所有数据的方法，但是，如果数据量少与10万条级别的话，一次读取，也是值得考虑的；

### jedis的sentinel连接池在spark Streaming中连接池无法释放的问题

我们一开始采用的是通过jedis的sentinel连接池的方法连接redis；
在连接哨兵模式的redis的时候，由于SparkStreaming程序中每个container并不会关闭，导致在spark的transform方法中，jedis sentinel 连接池在Streaming下出现连接池资源释放不了的bug，为了解决这个问题，我们弃用了jedis的sentinel连接池，每10秒手动询问redis的哨兵Master的地址，然后手动与redis进行连接，最终解决了这个问题。
手动实现jedis sentinel的连接的代码如下：

``` scala
// 由于redis sentinel 建立连接池不释放的坑，最后弃用连接池，自己实现寻找master的逻辑
val masterHostPort = getRedisMasterHostPortList(redisConf)
val masterHost = masterHostPort(0)
val masterPort = masterHostPort(1).toInt
...
// 得到master后，就可以直接建立redis连接了
val paRdd3: Iterator[(String, (String, String))] = paRdd2.flatMap { eachKV =>
。。。
	var jedis1: Jedis = null
	var redis1Val: String = null
	try {
	  jedis1 = new Jedis(masterHost, masterPort)
	  redis1Val = jedis1.hget(redisHashKey, mdn)
	} catch {
	  case e: Exception => {
	    printLog.error("jedis1 error：e" + e)
	  }
	} finally {
	  if (jedis1 != null) {
	    jedis1.close()
	  }
	}
}

```
询问哨兵Master在哪里的代码如下：

``` scala
/**
    * 得到redis sentinel 模式的master信息
    *
    * @param redisConf
    * @return
    */
  def getRedisMasterHostPortList(redisConf: RedisConf): util.List[String] = {
    printLog.debug("Trying to find master from available Sentinels...")
    //    @transient var masterFound: Boolean = false
    var masterFound: Boolean = false
    val sentinelList: Array[String] = redisConf.sentinels.split("\\|")
    for (sentinel <- sentinelList if !masterFound) {
      var jedis: Jedis = null
      try {
        jedis = new Jedis(sentinel.split(":")(0), sentinel.split(":")(1).toInt)
        val masterAddr: util.List[String] = jedis.sentinelGetMasterAddrByName(redisConf.masterName)
        // connected to sentinel...
        if (masterAddr == null || masterAddr.size != 2) {
          printLog.error("Can not get master addr, master name: " + redisConf.masterName + ". Sentinel: " + sentinel + ".")
        } else {
          masterFound = true
          printLog.debug("masterAddr: " + masterAddr)
          return masterAddr
        }
      } catch {
        case e: JedisException => {
          // resolves #1036, it should handle JedisException there's another chance
          // of raising JedisDataException
          printLog.error("Cannot get master address from sentinel running @ " + sentinel + ". Reason: " + e + ". Trying next one.")
        }
      } finally {
        if (jedis != null) {
          jedis.close()
        }
      }
    }
    return null
  }
```

### Redis的压力负载

后来我们对程序进行了加压测试，发现当压力增大后redis的主备切换非常频繁，原来在redis中，slaver会定时跟master通信，询问其是否健康，如果不健康，slaver就会强制进行主备切换，当redis访问频繁的时候，master来不及及时响应slaver的请求，就会导致主备切换频繁，解决这个问题的方法是修改询问时间，默认是 5s，适度调高就可以了。

当然redis还有一些其它优化参数，这里不做讨论；目前可以支持四千万的数据，

## 引用
https://github.com/xetorthio/jedis