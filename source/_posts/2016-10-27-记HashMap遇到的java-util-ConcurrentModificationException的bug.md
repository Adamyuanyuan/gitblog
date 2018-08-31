---
title: 记HashMap遇到的java.util.ConcurrentModificationException的bug
toc: true
date: 2016-10-27 10:22:51
tags: java
categories: java
---

#### 问题背景

spark Streaming 实时程序在联调期间稳定运行了两天，以为问题不大了，第二天早上的时候打开一看，竟然挂了，定位到代码，原来我的程序实时读取redis的数据为一个HashMap，直到挂的时候，Redis中数据一直在增大，共 6083条：

spark相关代码如下：

``` scala
 // 1. 从L1中删除过期的号码，同时Redis中的对应该K-V也删除
 ...
val sentinelPool = InternalRedisClient.getSentinelPool
var phoneSet: util.Map[String, String] = new util.HashMap[String, String]()
//            printLog.debug( "sentinelPool NumIdle0: " + sentinelPool.getNumIdle + " Active0: " + sentinelPool.getNumActive)

var jedis1: Jedis = null
try {
  jedis1 = sentinelPool.getResource
  //              printLog.debug( "sentinelPool NumIdle1: " + sentinelPool.getNumIdle + " Active1: " + sentinelPool.getNumActive)

  phoneSet = jedis1.hgetAll(redisHashKey)
  //              printLog.debug( "sentinelPool NumIdle3: " + sentinelPool.getNumIdle + " Active3: " + sentinelPool.getNumActive)

  printLog.debug("phoneSet_1: " + phoneSet)
} finally {
  if (jedis1 != null) {
    printLog.debug("close jedis1")
    jedis1.close()
  }
}

if (!phoneSet.isEmpty) { 
  for (eachPhoneKV: (String, String) <- phoneSet) { // 就在这里挂掉了
    val expirationDate: Int = eachPhoneKV._2.split("\\|")(4).toInt
    val today: Int = getNowDate.toInt
    if (today > expirationDate) {
      phoneSet.remove(eachPhoneKV._1)
      var jedis2: Jedis = null
      try {
        jedis2 = sentinelPool.getResource
        jedis2.hdel(redisHashKey, eachPhoneKV._1)
      } finally {
        if (jedis2 != null) {
          printLog.debug("close jedis2")
          jedis2.close()
        }
      }
    }
  }
  printLog.debug("phoneSet_filtedByData: " + phoneSet)
}
```
具体异常粘信息如下：
``` java
User class threw exception: org.apache.spark.SparkException: Job aborted due to stage failure: Task 4 in stage 0.0 failed 4 times, most recent failure: Lost task 4.3 in stage 0.0 (TID 170, NM-304-HW-XH628V3-BIGDATA-063): java.util.ConcurrentModificationException
at java.util.HashMap$HashIterator.nextEntry(HashMap.java:922)
at java.util.HashMap$EntryIterator.next(HashMap.java:962)
at java.util.HashMap$EntryIterator.next(HashMap.java:960)
at scala.collection.convert.Wrappers$JMapWrapperLike$$anon$2.next(Wrappers.scala:267)
at scala.collection.convert.Wrappers$JMapWrapperLike$$anon$2.next(Wrappers.scala:264)
at scala.collection.Iterator$class.foreach(Iterator.scala:727)
at scala.collection.AbstractIterator.foreach(Iterator.scala:1157)
at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
at scala.collection.TraversableLike$WithFilter.foreach(TraversableLike.scala:771)
at com.chinatelecom.bigdata.oidd2.Location$$anonfun$2$$anonfun$3.apply(Location.scala:871)
at com.chinatelecom.bigdata.oidd2.Location$$anonfun$2$$anonfun$3.apply(Location.scala:677)
```

#### 解决方案

项目太紧张，来不及详细分析java的源码了，根据经验redis中应该六千多条数据应该不是很大的，HashMap完全可以一次读取，从网上查到原因是因为remove操作导致的，在Iterator遍历过程中调用HashMap的remove方法会crash，有两个解决办法：

1. 一个解决办法是用一个ArrayList记录要删除的key,然后再遍历这个ArrayList,调用HashMap的remove方法以ArrayList的元素为key进行删除；这个方法需要额外的空间和时间，虽然也浪费的不多，但总感觉不够优雅；
2. 创建一个Iterator<Map.Entry<Integer, String>> iterator = map.entrySet().iterator();，然后用这个 iterator.remove()方法进行删除，这个是可以删除的；

我使用的是方法二，修改后的代码如下：
``` scala
var phoneMap: util.Map[String, String] = new util.HashMap[String, String]()
// printLog.debug( "sentinelPool NumIdle0: " + sentinelPool.getNumIdle + " Active0: " + sentinelPool.getNumActive)

var jedis1: Jedis = null
try {
  jedis1 = sentinelPool.getResource
  // printLog.debug( "sentinelPool NumIdle1: " + sentinelPool.getNumIdle + " Active1: " + sentinelPool.getNumActive)

  phoneMap = jedis1.hgetAll(redisHashKey)
  // printLog.debug( "sentinelPool NumIdle3: " + sentinelPool.getNumIdle + " Active3: " + sentinelPool.getNumActive)

  printLog.debug("phoneSet_1: " + phoneMap)
} finally {
  if (jedis1 != null) {
    printLog.debug("close jedis1")
    jedis1.close()
  }
}

// 1. 从L1中删除过期的号码，同时Redis中的对应该K-V也删除
if (!phoneMap.isEmpty) {
  val iterator: util.Iterator[Entry[String, String]] = phoneMap.entrySet().iterator()
  while (iterator.hasNext) {
    val eachPhoneKV: Entry[String, String] = iterator.next()
    val mdn = eachPhoneKV.getKey
    val redisValue = eachPhoneKV.getValue

    var expirationDate: Int = 0
    try {
      expirationDate = redisValue.split("\\|")(4).toInt
      val today: Int = getNowDate.toInt
      if (today > expirationDate) {
        printLog.info("delete this data for expirationDate:" + eachPhoneKV)
        iterator.remove()
        // phoneMap.remove(mdn) // 这一句是错误的，因为无法据此删除
        var jedis2: Jedis = null
        try {
          jedis2 = sentinelPool.getResource
          jedis2.hdel(redisHashKey, mdn)
        } finally {
          if (jedis2 != null) {
            printLog.debug("close jedis2")
            jedis2.close()
          }
        }
      }
    } catch {
      // 如果解析发生异常，则redis中删掉这个key，并且在phoneMap中同时删除
      case ex: Exception => {
        printLog.error("redis error data and del it: " + eachPhoneKV + " error: " + ex)
        iterator.remove()
        var jedis4: Jedis = null
        try {
          jedis4 = sentinelPool.getResource
          jedis4.hdel(redisHashKey, mdn)
        } finally {
          if (jedis4 != null) {
            printLog.debug("close jedis4")
            jedis4.close()
          }
        }
      }
    }
  }
```
然后测试，打包，部署，OK，解决。

#### 原理说明

遍历HashMap有三种方法，分别是: 
``` java
for(Map.Entry<Integer, String> entry : map.entrySet()){}  // scala中为 <-

for(Integer key : keySet){}

Iterator<Map.Entry<Integer, String>> it = map.entrySet().iterator();
        while(it.hasNext()){}
```
其实上面的三种遍历方式从根本上讲都是使用的迭代器，之所以出现不同的结果是由于remove操作的实现不同决定的。

首先前两种方法都在调用nextEntry方法的同一个地方抛出了异常，虽然remove成功了，但是在迭代器遍历下一个元素的时候抛出异常：
``` java
final Entry<K,V> nextEntry() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    Entry<K,V> e = next;
    ...
}
```
这里modCount是表示map中的元素被修改了几次(在移除，新加元素时此值都会自增)，而expectedModCount是表示期望的修改次数，在迭代器构造的时候这两个值是相等，如果在遍历过程中这两个值出现了不同步就会抛出ConcurrentModificationException异常。

1、HashMap的remove方法实现
``` java
public V remove(Object key) {
    Entry<K,V> e = removeEntryForKey(key);
    return (e == null ? null : e.value);
}
```

2、HashMap.KeySet的remove方法实现
``` java
public boolean remove(Object o) {
    return HashMap.this.removeEntryForKey(o) != null;
}
```

3、HashMap.HashIterator的remove方法实现
``` java
public void remove() {
   if (current == null)
        throw new IllegalStateException();
   if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
   Object k = current.key;
   current = null;
   HashMap.this.removeEntryForKey(k);
   expectedModCount = modCount;
}
```
以上三种实现方式都通过调用HashMap.removeEntryForKey方法来实现删除key的操作。在removeEntryForKey方法内只要移除了key modCount就会执行一次自增操作，此时modCount就与expectedModCount不一致了，上面三种remove实现中，只有第三种iterator的remove方法在调用完removeEntryForKey方法后同步了expectedModCount值与modCount相同，所以在遍历下个元素调用nextEntry方法时，iterator方式不会抛异常。

``` java
final Entry<K,V> removeEntryForKey(Object key) {
    int hash = (key == null) ? 0 : hash(key.hashCode());
    int i = indexFor(hash, table.length);
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            if (prev == e)
                table[i] = next;
            else
                prev.next = next;
            e.recordRemoval(this);
            return e;
        }
        prev = e;
        e = next;
    }

    return e;
}
```

#### 其它思考

1、如果是遍历过程中增加或修改数据呢？
增加或修改数据只能通过Map的put方法实现，在遍历过程中修改数据可以，但如果增加新key就会在下次循环时抛异常，因为在添加新key时modCount也会自增。

2、有些集合类也有同样的遍历问题，如ArrayList，通过Iterator方式可正确遍历完成remove操作，直接调用list的remove方法就会抛异常。
``` java
//会抛ConcurrentModificationException异常
for(String str : list){
	list.remove(str);
}

//正确遍历移除方式
Iterator<String> it = list.iterator();
while(it.hasNext()){
	it.next();
	it.remove();
}
```

3、jdk为什么这样设计，只允许通过iterator进行remove操作？

HashMap和keySet的remove方法都可以通过传递key参数删除任意的元素，而iterator只能删除当前元素(current);
对于通过HashMap的remove方法来说，一旦删除的元素是iterator对象中next所正在引用的，如果没有通过modCount与 expectedModCount的比较实现快速失败抛出异常，下次循环该元素将成为current指向，此时iterator就遍历了一个已移除的过期数据，所以一定要判断这两个值是否一致。

4、这是一个坑，如果IDE能提示就好了，下次注意

#### 引用

http://afredlyj.github.io/posts/hashmap-concurrentmodificationexception.html
http://dumbee.net/archives/41
http://blog.csdn.net/wzy_1988/article/details/51423583