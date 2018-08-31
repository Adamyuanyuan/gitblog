---
title: Spark踩坑之Streaming程序实时读取数据库
toc: false
date: 2016-11-24 10:51:50

tags: 
- spark streaming
- mysql
- spark开发
categories: spark开发
---

## 背景
一个SparkStreaming的项目，由于需要从Mysql数据库中实时读取一些信息，然后生成特定的数据结构进行动态的处理，在此过程中踩了一些坑，谨记：

#### 初始方案

在rdd的每个partition中创建一个内部数据库连接池单例对象InternalMDBManager，然后使用一个连接池

``` scala
val dStream2: DStream[(String, (String, String))] = dStream1.transform {
  rdd1 => {
    val numPartitions = rdd1.getNumPartitions
    printLog.info("numPartitions: " + numPartitions)

    val rdd3: RDD[(String, (String, String))] = rdd1.mapPartitions {
      paRdd1 => {


        /**
          * Author: wangxiaogang
          *
          * 由于需要在spark中分布式读取Mysql的数据，所以需要创建可以分布式分发的单例对象连接池
          */
        object InternalMDBManager extends Serializable {
          @transient private var pool: ComboPooledDataSource = _

          /**
            * 从连接池获取连接
            *
            * @return
            */
          def getConnection: Connection = {
            try {
              pool.getConnection()
            } catch {
              case ex: Exception => ex.printStackTrace()
                null
            }
          }

          def closeConnection(connection: Connection): Unit = {
            if (!connection.isClosed) connection.close()
          }

          /**
            * 创建连接池
            *
            * @param sqlPoolConf
            */
          def makePool(sqlPoolConf: SqlPoolConf): Unit = {
            if (pool == null) {
              try {
                pool = new ComboPooledDataSource(true)
                pool.setJdbcUrl(sqlPoolConf.jdbcUrl)
                pool.setDriverClass(sqlPoolConf.driverClass)
                pool.setUser(sqlPoolConf.user)
                pool.setPassword(sqlPoolConf.password)
                pool.setMaxPoolSize(sqlPoolConf.maxPoolSize)
                pool.setMinPoolSize(sqlPoolConf.minPoolSize)
                pool.setAcquireIncrement(sqlPoolConf.acquireIncrement)
                pool.setInitialPoolSize(sqlPoolConf.initialPoolSize)
                pool.setMaxIdleTime(sqlPoolConf.maxIdleTime)
              } catch {
                case ex: Exception => ex.printStackTrace()
              }
            }
          }
        }

        InternalRedisClient.makeSentinelPool(redisConf)

        InternalMDBManager.makePool(sqlPoolConf)
        val sqlConn = sqlPool.getConnection
        val sqlConn = InternalMDBManager.getConnection

        。。。
      }
    }
  }
}
```

这个方法听起来是极好的，在每个jvm中创建一个连接池，然后不同的批数据使用共同的连接池，但是在实践的过程中发现，在foreachRDD中使用这种方法是可以的，网上有很多类似的例子；
但是在transform方法中，如果在map，flatMap，filter等方法外面建立连接池，会出现连接池无法释放的问题，无论你如何使用finally释放，都释放不了；
解决方法是**避免使用连接池，将数据库建立与释放操作封装到同一个函数里**，在我的问题里，因为对数据库的操作不会很频繁，所以不需要引入连接池，这样将会及时释放数据库资源。

最终方案是如下：
``` scala
def getPositionSubDataMap(sqlPoolConf: SqlPoolConf): util.HashMap[Int, util.LinkedList[PositionSubData]] = {
val currentTime: Long = getCurrentTime
// todo: 如果数据量比较大的话，判断时间语句直接放到mysql查询的时候
val sqlStr =
"""some sqls"""

// 这里告诉我们，写代码的时候不要盲目建立资源池，不要简单的东西复杂化
val url = sqlPoolConf.jdbcUrl
val user = sqlPoolConf.user
val password = sqlPoolConf.password
var sqlConn: Connection = null
//    var pstmt: PreparedStatement = null
//    var rs: ResultSet = null
val positionSubDataMap: util.HashMap[Int, util.LinkedList[PositionSubData]] = new util.HashMap[Int, util.LinkedList[PositionSubData]]()
try {
  sqlConn = DriverManager.getConnection(url, user, password)
  val pstmt: PreparedStatement = sqlConn.prepareStatement(sqlStr)
  val rs: ResultSet = pstmt.executeQuery()
  //    var positionSubDataList = ArrayBuffer[PositionSubData]

  while (rs.next()) {
    val subId: Long = rs.getLong("sub_id")
    val spId: String = rs.getString("sp_id")
    val locationId: String = rs.getString("location_id")
    val provId: String = rs.getString("prov_id")
    val cityCode: String = rs.getString("city_code")
    val intervalTime: Int = rs.getInt("interv")
    val available: Int = rs.getInt("available")
    val startTime = rs.getLong("start_time")
    val endTime = rs.getLong("end_time")
    val centerLongitude: Double = rs.getDouble("center_longitude")
    val centerLatitude: Double = rs.getDouble("center_latitude")
    val radius: Int = rs.getInt("radius")
    val shape: String = rs.getString("shape")

    //      printLog.info("cityCode:" + cityCode)

    val positionSubData: PositionSubData = PositionSubData(subId, spId, locationId, provId, cityCode, intervalTime,
      available, startTime, endTime, centerLongitude, centerLatitude, radius, shape)
    //      printLog.info("positionSubData: " + positionSubData)

    if ((startTime < currentTime) && (endTime > currentTime)) {
      //        positionSubDataList. += positionSubData
      val cityCodeArray: Array[String] = cityCode.split("\\|")
      for (eachCityCode <- cityCodeArray) {
        if (positionSubDataMap.get(eachCityCode.toInt) == null) {
          // 代表以该城市为key没有其它景区
          val positionSubDataList: util.LinkedList[PositionSubData] = new util.LinkedList[PositionSubData]
          positionSubDataList.add(positionSubData)
          //            printLog.info("1positionSubDataList：" + positionSubDataList)
          positionSubDataMap.put(eachCityCode.toInt, positionSubDataList)
          //            printLog.info("1positionSubDataMap: " + positionSubDataMap)
        } else {
          // 代表以该城市为key有其它景区，并且已经记录在案
          val positionSubDataList: util.LinkedList[PositionSubData] = positionSubDataMap.get(eachCityCode.toInt)
          positionSubDataList.add(positionSubData)
          //            printLog.info("2positionSubDataList：" + positionSubDataList)
          //            printLog.info("2eachCityCode.toInt:" + eachCityCode.toInt)
          positionSubDataMap.put(eachCityCode.toInt, positionSubDataList)
        }
      }
    }
  }
  if (rs != null) {
    rs.close()
  }
  if (pstmt != null) {
    pstmt.close()
  }
} catch {
  case e: Exception => {
    printLog.error("数据库连接错误：e" + e)
  }
} finally {
  if (sqlConn != null) {
    sqlConn.close()
  }
}
positionSubDataMap
}
```


## 总结

综上所述，Streaming程序中动态连接数据库要谨慎，要及时查看数据库的连接状态，看看数据库连接有没有被及时释放，它不会马上就报错，但随着连接数到达数据库的最高值的时候就会出错，检测不及时，等上了生产再出问题，就后悔莫及。

## 参考链接
在foreachRDD中建立连接池的例子
http://www.cnblogs.com/xlturing/p/spark.html