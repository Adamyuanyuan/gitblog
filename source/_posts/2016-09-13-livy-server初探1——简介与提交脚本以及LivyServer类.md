---
title: livy-server初探1——简介与提交脚本以及LivyServer类
date: 2016-09-13 09:51:25
toc: true
tags:
- spark
- livy
categories: spark
---

[Livy server](http://livy.io/)是针对Spark的开源的REST接口，使得我们可以通过REST接口来实现与Spark交互,之前应该是Hue框架的一个功能模块，现在已经独立出来啦。具有如下功能：
1） 可以与scala、python、R shell客户端交互，执行一些代码片段
2） 可以提交整个Spark Job,支持scala、python、java编写的Spark job。

## Welcome to Livy

下面是官网文档中我对 Welcome to Livy的翻译：

Livy通过提供REST服务来简化与Spark集群的交互。它可以通过job或者代码片段的方式来提交Spark任务，并同步或者异步地获得任务的结果，以及管理spark context，上述功能通过简单的REST接口或者RPC服务来实现。livy也可以简化Spark与一些应用程序之间的交互，使得Spark可以用于一些web应用(比如Hue)。更多的功能包括：

- 拥有长期运行的Spark Contexts供多用户提交各种的Spark job；
- 不同的任务和用户可以共享cached RDD或者DataFrames；
- 多个SC可以按计划同时运行，为了使得SC具有更好的容错性和并发性，可以将SC运行在yarn/Mesos等集群中；
- 可以通过java/scala客户端的API来提交预编译好的jar包或代码片段
- 支持一定的安全机制
- Apache-licensed 100%开源

与ReadMe中的文档结合再补充几条：

- 支持Scala，Python，R Shell的交互；
- 支持 Scala，Java，Python的批量提交；
- 不需要你对你自己的代码增加任何改变；

官网和github逛了一整子后不禁感叹，新东西总是缺乏底层的文档的，所以要了解它就要阅读源码了。

## 从./bin/livy-server进入
``` bash 
usage="Usage: livy-server (start|stop)"

# 指定LIVY_HOME与LIVY_CONF_DIR，上述 `export LIVY_HOME=$(cd $(dirname $0)/.. && pwd)`这种写法值得学习，代表将LIVY_HOME环境变量设为本脚本的父目录，通过这种写法，增强了脚本的可移植性，另外注明一点，dirname这个命令在命令行里是不能用的，只有写在脚本中才能起作用。
export LIVY_HOME=$(cd $(dirname $0)/.. && pwd)
LIVY_CONF_DIR=${LIVY_CONF_DIR:-"$LIVY_HOME/conf"}

# 运行所有的livy-env.sh中的环境变量，并使用set -a 表示输出所有的环境变量的改变
if [ -f "${LIVY_CONF_DIR}/livy-env.sh" ]; then
  # Promote all variable declarations to environment (exported) variables
  set -a
  . "${LIVY_CONF_DIR}/livy-env.sh"
  set +a
fi
```

接下来可以看到调用了 start_livy_server，以及stop的代码(其实就是ps -p到livy的那个进程，然后kill掉，值得借鉴)：

``` bash
option=$1

case $option in

  (start)
    start_livy_server "new"
    ;;

  ("")
    # make it compatible with previous version of livy-server
    start_livy_server "old"
    ;;

  (stop)
    if [ -f $pid ]; then
      TARGET_ID="$(cat "$pid")"
      if [[ $(ps -p "$TARGET_ID" -o comm=) =~ "java" ]]; then
        echo "stopping livy_server"
        kill "$TARGET_ID" && rm -f "$pid"
      else
        echo "no livy_server to stop"
      fi
    else
      echo "no livy_server to stop"
    fi
    ;;

  (*)
    echo $usage
    exit 1
    ;;

esac
```
接下来就是start_livy_server函数了，它做了下面几件事情：

1. 找到livy的jar包；
2. 设置LIVY_CLASSPATH并将SPARK与HADOOP以及YARN的CONF_DIR加入到classpath中；
3. 如果是`./bin/livy-server`启动的程序，就直接运行 "$RUNNER $LIVY_SERVER_JAVA_OPTS -cp $LIVY_CLASSPATH:$CLASSPATH com.cloudera.livy.server.LivyServer"
4. 如果是`./bin/livy-server start`启动的程序，则增加了日志记录，以方便查看，所以推荐新版本使用带start参数的方式

## com.cloudera.livy.server.LivyServer

从后面进入：原来是创建了一个 LivyServer的server，然后start和join启动

``` scala LivyServer.scala
object LivyServer {

  def main(args: Array[String]): Unit = {
    val server = new LivyServer()
    try {
      server.start()
      server.join()
    } finally {
      server.stop()
    }
  }

}
```
### LiveServer的属性

LivyServer的属性不多，（与spark源码相比）：

``` scala LivyServer.scala
import LivyConf._

  private var server: WebServer = _
  private var _serverUrl: Option[String] = None
  // make livyConf accessible for testing
  private[livy] var livyConf: LivyConf = _

  private var kinitFailCount: Int = 0
  private var executor: ScheduledExecutorService = _
```

### start()函数
然后是start()函数了
首先，从配置文件中读取配置信息（这一块内容自己写得时候可以借用）：

- 从配置信息中得到host和port

``` scala LivyServer.scala
livyConf = new LivyConf().loadFromFile("livy.conf")
val host = livyConf.get(SERVER_HOST)
val port = livyConf.getInt(SERVER_PORT)
# 这个而没有看懂
val multipartConfig = MultipartConfig(
    maxFileSize = Some(livyConf.getLong(LivyConf.FILE_UPLOAD_MAX_SIZE))
  ).toMultipartConfigElement
```

- 测试SparkHome是否设置成功

如下代码，这里使用了require方法对参数进行先决条件检测(值得借鉴)
``` scala LivyServer.scala
// Make sure the `spark-submit` program exists, otherwise much of livy won't work.
testSparkHome(livyConf)
...

/**
* Sets the spark-submit path if it's not configured in the LivyConf
*/
private[server] def testSparkHome(livyConf: LivyConf): Unit = {
val sparkHome = livyConf.sparkHome().getOrElse {
  throw new IllegalArgumentException("Livy requires the SPARK_HOME environment variable")
}

require(new File(sparkHome).isDirectory(), "SPARK_HOME path does not exist")
}
```

- 测试spark-submit命令可用，否则livy无法工作(值得借鉴):

这里的代码写得太精彩了！先是定义一个`$SPAKR_HOME/bin/spark-sumbit --version`的命令，使用java的ProcessBuilder，然后可以得到exitCode和重定向的标准输出结果，如果结果是"version ..."的话，就代表执行成功，输出结果；然后对这个version进行正则匹配，如果是1.6到2.0版本之间，就返回true，否则就说明spark版本不支持；

``` scala LivyServer.scala
testSparkSubmit(livyConf)
...

/**
* Test that the configured `spark-submit` executable exists.
*
* @param livyConf
*/
private[server] def testSparkSubmit(livyConf: LivyConf): Unit = {
try {
  testSparkVersion(sparkSubmitVersion(livyConf))
} catch {
  case e: IOException =>
    throw new IOException("Failed to run spark-submit executable", e)
}
}

...
/**
* Return the version of the configured `spark-submit` version.
*
* @param livyConf
* @return the version
*/
private def sparkSubmitVersion(livyConf: LivyConf): String = {
val sparkSubmit = livyConf.sparkSubmit()
val pb = new ProcessBuilder(sparkSubmit, "--version")
pb.redirectErrorStream(true)
pb.redirectInput(ProcessBuilder.Redirect.PIPE)

if (LivyConf.TEST_MODE) {
  pb.environment().put("LIVY_TEST_CLASSPATH", sys.props("java.class.path"))
}

val process = new LineBufferedProcess(pb.start())
val exitCode = process.waitFor()
val output = process.inputIterator.mkString("\n")

val regex = """version (.*)""".r.unanchored

output match {
  case regex(version) => version
  case _ =>
    throw new IOException(f"Unable to determine spark-submit version [$exitCode]:\n$output")
}
}

/**
* Throw an exception if Spark version is not supported.
* @param version Spark version
*/
private[server] def testSparkVersion(version: String): Unit = {
val versionPattern = """(\d)+\.(\d)+(?:\.\d*)?""".r
// This is exclusive. Version which equals to this will be rejected.
val maxVersion = (2, 0)
val minVersion = (1, 6)

val supportedVersion = version match {
  case versionPattern(major, minor) =>
    val v = (major.toInt, minor.toInt)
    v >= minVersion && v < maxVersion
  case _ => false
}
require(supportedVersion, s"Unsupported Spark version $version.")
}
```

- 通过livy.spark.master是否以yarn开头判断是否需要初始化YarnClient

``` scala LivyServer.scala
// Initialize YarnClient ASAP to save time.
if (livyConf.isRunningOnYarn()) {
  Future { SparkYarnApp.yarnClient }
}

// SparkYarnApp.scala
// YarnClient is thread safe. Create once, share it across threads.
lazy val yarnClient = {
  val c = YarnClient.createYarnClient() // 这里调用的是yarn提供的API
  c.init(new YarnConfiguration())
  c.start()
  c
}
```

- 接下来启动WebServer

这个webServer是Jetty的WebServer，通过设置是否使用ssl的配置来判断启动的是http server还是 https server，也会判断有没有Kerberos，设置IP，端口，日志等。（以后写得时候得查看Jetty的API和文档）

``` scala LivyServer.scala
server = new WebServer(livyConf, host, port)
server.context.setResourceBase("src/main/com/cloudera/livy/server")
server.context.addEventListener(
  new ServletContextListener() with MetricsBootstrap with ServletApiImplicits {

    private def mount(sc: ServletContext, servlet: Servlet, mappings: String*): Unit = {
      val registration = sc.addServlet(servlet.getClass().getName(), servlet)
      registration.addMapping(mappings: _*)
      registration.setMultipartConfig(multipartConfig)
    }

    override def contextDestroyed(sce: ServletContextEvent): Unit = {

    }

    override def contextInitialized(sce: ServletContextEvent): Unit = {
      try {
        val context = sce.getServletContext()
        context.initParameters(org.scalatra.EnvironmentKey) = livyConf.get(ENVIRONMENT)
        mount(context, new InteractiveSessionServlet(livyConf), "/sessions/*")
        mount(context, new BatchSessionServlet(livyConf), "/batches/*")
        context.mountMetricsAdminServlet("/")
      } catch {
        case e: Throwable =>
          error("Exception thrown when initializing server", e)
          sys.exit(1)
      }
    }

  })
```

