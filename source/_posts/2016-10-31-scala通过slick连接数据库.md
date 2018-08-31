---
title: scala通过slick连接数据库
toc: true
date: 2016-10-31 19:20:33
tags: 
- spark
- scala
- 持续更新
categories: 
- spark开发
---
（持续更新）
由于Spark是由scala语言开发的，scala语言可以使用到所有java语言中的特性，所以spark连接数据库（比如Mysql）有很多种方法，这里记录两种我使用到的高级用法以及一些教训，分别是：
1. 使用Slick优雅地连接数据库；
2. 如何使用SparkStreaming实时地获取数据库中的内容；
3. 连接数据库过程中的踩坑集锦。

## 使用Slick优雅地连接数据库

如果使用scala语言，当然可以想到的是，通过java连接数据库的方式连接数据库是没有问题的，但是scala语言有没有自己更加优雅地方法连接数据库呢？答案是肯定的，非常推荐使用：Slick

### Slick简介
Slick 是 TypeSafe 推出的 Scala 数据库访问库。开发者可以使用 Scala 语言风格来编写数据查询，而不是用 SQL 。 Slick 对于 Scala 来说，有如 LINQ 至于 C#，或者类似于其它平台上的 ORM 系统，它使用应用使用数据库有如使用 Scala 内置的集合类型（比如列表，集合等）一样方便。当然如有需要你还是可以直接使用 SQL 语句来查询数据库。
使用 Slick 而不直接使用 SQL 语句，可以使用编译器帮助发现一些类型错误，同时 Slick 可以为不同的后台数据库类型生成查询。它具有一些如下的特性：

1. Scala 

所有查询，表格和字段映射，以及类型都采用普通的 Scala 语法。
``` scala
class Coffees(tag: Tag) extends Table[(String, Double)](tag, "COFFEES") {
    def name = column[String]("COF_NAME", O.PrimaryKey)
    def price = column[Double]("PRICE")
    def * = (name, price)
}
val coffees = TableQuery[Coffees]
```
数据访问接口类型 Scala 的集合类型
``` scala
// Query that only returns the "name" column
coffees.map(_.name)

// Query that does a "where price < 10.0"
coffees.filter(_.price < 10.0)
```

2. 类型安全

你使用的 IDE 可以帮助你写代码 在编译时而无需到运行时就可以发现一些错误
``` scala
// The result of "select PRICE from COFFEES" is a Seq of Double
// because of the type safe column definitions
val coffeeNames: Seq[Double] = coffees.map(_.price).list

// Query builders are type safe:
coffees.filter(_.price < 10.0)
// Using a string in the filter would result in a compilation error
```

3. 可以组合

查询接口为函数，这些函数可以多次组合和重用。可以使用函数式的方式来访问数据库

``` scala
// Create a query for coffee names with a price less than 10, sorted by name
coffees.filter(_.price < 10.0).sortBy(_.name).map(_.name)
// The generated SQL is equivalent to:
// select name from COFFEES where PRICE < 10.0 order by NAME
```

4. 支持几乎所有常见的数据库

	- DB2 (via slick-extensions)
	- Derby/JavaDB
	- H2
	- HSQLDB/HyperSQL
	- Microsoft Access
	- Microsoft SQL Server (via slick-extensions)
	- MySQL
	- Oracle (via slick-extensions)
	- PostgreSQL
	- SQLite

对于其它的一些数据库类型 Slick 也提供了有限的支持。

关于Slick的具体教程以及API，可以参阅 
[Slick官网](http://slick.lightbend.com/)
[极客学院Slick中文教程](http://wiki.jikexueyuan.com/project/slick-guide/)
以及google

### 如何使用到Spark项目中
以我自己摸索的方法为例，当然会有更多方法

#### 配置文件中配置数据库连接信息
Slick默认会读取项目顶层的配置文件，当然配置文件的路径可以手动指定，默认在顶层的配置文件路径下，我的配置文件在 `resources/application.conf`中：

``` conf resources/application.conf
// todo: 使其变成可以从外部文件载入
// 开发环境Mysql
mysql = {
  url = "jdbc:mysql://someIp:3306/someDb?useUnicode=true&characterEncoding=utf-8"
  driver = "com.mysql.jdbc.Driver"
  connectionPool = disabled
  keepAliveConnection = true
  databaseName = "someDb"
  user = "user"
  password = "password"
}
```
#### 定义数据库表格对应的case class

比如：该AppFrame是我定义的一个应用框架的样例类，与数据库中的字段有对应关系
PS：AppFrame的设置是为了将一切配置写到数据库中，这样可以实现项目的热切换，一套程序可以不需要重新编译而使用到不同的环境不同的策略中，亲测有效。

``` scala bean/AppFrame.scala
/**
  * Author: wangxiaogang
  * Date: 2016/10/1
  * Email: wangxiaogang@chinatelecom.cn
  * 应用的整体描述，包括 appId, app名称，输入类型，输入类型详细配置表，输出类型，输出类型详细配置表
  */
case class AppFrame (
               id: Int,
               name: String,
               inputStr: String,
               // 代表对应的输入在所在类型的输入中的id
               inputId: Int,
               outputStr: String,
               outId: Int,
               redisStr: String,
               redisId: Int,
               sqlPoolStr: String,
               sqlPoolId:Int
               ){}
```

#### 使用Slick编写读取数据库的逻辑

这里就是slick的优势，非常简单

``` scala dao/MysqlDao.scala
/**
  * Author: wangxiaogang
  * Date: 2016/9/29
  * Email: wangxiaogang@chinatelecom.cn
  * 与Mysql数据库的交互类，使用了Slick方式
  */
object MysqlDao {
  /**
    * 通过appId获取到应用的整体描述，包括 appId, 输入类型，输入类型详细配置表，输出类型，输出类型详细配置表
    * 这里简单试验大数据实时处理框架的构思是否可行
    *
    * @param appId
    * @return
    * @todo : 目前只配置输入类型与输出类型的配置，以后尽量把所有数据处理方案都能以配置的形式写入数据库中
    */
  def getAppFrame(appId: Int): AppFrame = {
    implicit val getResult = GetResult(r =>
      AppFrame(r.nextInt(), r.nextString(), r.nextString(), r.nextInt(), r.nextString(), r.nextInt(), r.nextString(),
        r.nextInt(),  r.nextString(), r.nextInt()))
    val q = sql"""SELECT * FROM appframe WHERE id = $appId""".as[AppFrame]

    val db = Database.forConfig("mysql")
    try {
      val fu = db.run(q)
      Await.result(fu, 10 seconds).head
    } finally {
      db.close()
    }
  }
  ...
}
```
如上，就这么简单几行，就能连接数据库了，并且将其转化为对应的样例类，是不是超级好用
当然，我只是使用了一点皮毛，它还有很多有用的特性我没有使用到。

## 引用
http://slick.lightbend.com/
http://wiki.jikexueyuan.com/project/slick-guide/
