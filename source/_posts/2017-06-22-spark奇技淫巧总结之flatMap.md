---
title: spark奇技淫巧总结之强大的flatMap
toc: false
date: 2017-06-22 19:00:17
tags: spark开发
categories: spark开发
---

spark RDD与DStream API支持很多好用的算子，最常用的莫过于map和filter了，顾名思义可知：
**map**： 返回一个新的分布式数据集，其中每个元素都是由源RDD中一个元素经func转换得到的；
**filter**： 返回一个新的数据集，其中包含的元素来自源RDD中元素经func过滤后（func返回true时才选中）的结果；

举个例子：如下RDD a 的一个partition有10个元素，那么：

``` scala

val a: RDD[String] = sc.parallelize(Seq("1", "2", "1", "1", "4", "3", "3", "1", "5", "6"))
// 后的结果肯定有 10个元素
val b: RDD[String] = a.map(func1()) 
// 后的结果肯定 <= 10个元素，并且元素的内容不会改变
val c: RDD[String] = a.filter(func2()) 

```
通过类似如上样例代码，可知，如果你要转化内容，通过map，10个元素变化后还是10个元素，如果你想过滤内容，c 通过a过滤后，虽然一些值被过滤掉了，**但是没被过滤掉的值依然没有变化**。

先看flatMap的spark官网文档的官方解释： 
**类似于map，但每个输入元素可以映射到0到n个输出元素（所以要求func必须返回一个Seq而不是单个元素）。**
RDD API的解释：
**Return a new RDD by first applying a function to all elements of this RDD, and then flattening the results.**

无论网上的一些例子还是解释，都给人一种flatMap只能把数据“压扁”的感觉。其实flatMap不仅仅能把数据压扁，还能把数据“拔高”，还能把数据“过滤”。flatMap不像map，只能1对1，也不像filter，只能1对1或0(还不能改变数据)，它是1对n(自然数)的。

### 解决边处理边过滤的需求

我在开发中遇到过很多次这种需求：**既要改变内容，同时不符合要求的数据需要过滤**，这种情况下该怎么办呢？

#### 简单方案
先filter，然后map，或者先map，然后filter，这样都可以完成这种需求，但是它有以下问题：
1. 重复计算了，map和filter中很多类似的逻辑都要多算一遍，这在大量数据集下是不可容忍的；
2. 对于有些判断，只可能判断一次，第二次计算结果会不一样，比如在transform中需要与外界redis等交互的判断，这种情况下，结果都是错误的；

这种情况下，就可以用flatMap来解决，嘿嘿~

直接说解决方案吧：对数据进行flatMap进行转换，如果不符合要求要过滤，则直接返回 None即可，样例代码如下：

``` scala
@RunWith(classOf[JUnitRunner])
class FlatMapTest extends FunSuite with Matchers {
  /**
    * 测试 rdd FlatMapTest 的过滤用法是否可行
    */
  test("FlatMapTest should work") {
    println("FlatMapTest  started")
    val conf = new SparkConf().setAppName("Simple Application").setMaster("local")
    val sc = new SparkContext(conf)
    val a: RDD[String] = sc.parallelize(Seq("1", "2", "1", "1", "4", "3", "3", "1", "5", "6"))
    val b: RDD[(String, String)] = a.map(f => (f, f + "asd"))
    val c = b.mapPartitions {
      eachPar => {
        eachPar.flatMap(f =>
          if (f._2.startsWith("3")) {
            Some(f._1, f._2 + "--b")
          } else {
            None
          }
        )
      }
    }
    c.cache()
    c.foreach(println(_))
    println(c.count())
    c.unpersist()
    println("FlatMapTest ended")
  }
}

```
上述的代码就可以做到处理的过程中进行过滤了~

### 解决一条数据对应多条数据的需求

我在开发中也遇到过这种需求：**RDD中一个元素处理后可能会变成多个元素**，比如一个用户可能会同时在多个景区存在，为了便于统计和输出，需要同时输出多个，这种情况下可以用flatMap来解决：

首先看看RDD flatMap的定义：

``` scala
  /**
   *  Return a new RDD by first applying a function to all elements of this
   *  RDD, and then flattening the results.
   */
  def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))
  }

```
它底层调用的是 Iterator 的flatMap
``` scala
  /** Creates a new iterator by applying a function to all values produced by this iterator
   *  and concatenating the results.
   *
   *  @param f the function to apply on each element.
   *  @return  the iterator resulting from applying the given iterator-valued function
   *           `f` to each value produced by this iterator and concatenating the results.
   *  @note    Reuse: $consumesAndProducesIterator
   */
  def flatMap[B](f: A => GenTraversableOnce[B]): Iterator[B] = new AbstractIterator[B] {
    private var cur: Iterator[B] = empty
    def hasNext: Boolean =
      cur.hasNext || self.hasNext && { cur = f(self.next).toIterator; hasNext }
    def next(): B = (if (hasNext) cur else empty).next()
  }
```

可以看到，flatMap的返回也是 Iterator[B]，所以，只要我们以 Iterator 的方式返回，就可以返回多条数据了，在Scala中，只要你返回的格式是某种可以Iterator的，就满足要求了，在Scala中，所有的集合Iterable都是 trait Iterator的一种扩展，无论是 seq set 还是map，这就很方便了，样例代码如下：

``` scala
  test("FlatMapTest should work") {
    println("FlatMapTest  started")
    val conf = new SparkConf().setAppName("Simple Application").setMaster("local[3]")
    val sc = new SparkContext(conf)
    val a: RDD[String] = sc.parallelize(Seq("1", "2", "1", "1", "4", "3", "3", "1", "5", "6", "5"))
    val b: RDD[(String, String)] = a.map(f => (f, f + "asd"))
    val c = b.mapPartitions {
      eachPar => {
        eachPar.flatMap(f => {
          val returnSeq = ArrayBuffer.empty[(String, String)]
          if (f._2.startsWith("3")) {
            returnSeq += ((f._1, f._2 + "--b"))
            returnSeq += ((f._1, f._2 + "-4c"))
            returnSeq += ((f._2, f._1 + "-6d"))
//            Seq(Some(f._1, f._2 + "--b"), Some(f._1, f._2 + "--c"), Some(f._2, f._1 + "--d"))
          } else {
          }
          returnSeq
        }
        )
      }
    }
    c.cache()
    c.foreach(println(_))
    println(c.count())
    c.unpersist()
    println("FlatMapTest ended")
  }
```

当然，实际处理比这复杂多了，你可以在flatMap中随意发挥，进行各种对外的连接查询操作。