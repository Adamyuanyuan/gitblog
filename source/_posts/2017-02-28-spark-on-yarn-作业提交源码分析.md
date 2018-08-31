---
title: spark on yarn 作业提交源码分析
toc: true
date: 2017-02-28 19:08:16
tags:
- spark
- yarn
categories: spark
---

最近因为上线hdfs的federation功能，测试spark程序的时候遇到了问题，在分析此问题的过程中对spark on yarn提交作业的过程记录一下，以SparkPi为例，从spark-submit开始，通过debug日志分析详细的过程：
整个过程中主要涉及：spark源码（1.6.2）hadoop源码（2.6.0-cdh5.4.7） Kerberos相关
#### 从spark-sumbit开始
提交命令：
``` bash
spark-submit --master yarn-client --class SparkPi ./testJars/my.jar
```
进入`./bin/spark-submit`
``` bash
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
以上脚本调用了spark-class脚本，并增加了参数 org.apache.spark.deploy.SparkSubmit。
进入 `./bin/spark-class`

``` bash
// 首先执行load-spark-env.sh载入 ./conf/spark-env.sh，保证只载入一次，如果之前手动载入过一次的话，就不会再覆盖载入
. "${SPARK_HOME}"/bin/load-spark-env.sh
// 寻找spark安装包，是这个样子：`spark-assembly.*hadoop.*\.jar$`，而且只能有一个
...
// 最后会通过java调用spark的 `org.apache.spark.launcher.Main`作为spark应用程序的主入口，首先循环读取ARG参数，加入到CMD中：
CMD=()
while IFS= read -d '' -r ARG; do
  CMD+=("$ARG")
done < <("$RUNNER" -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@")
exec "${CMD[@]}"
```
翻译过来就是：
```
/bin/java -cp /usr/lib/spark/lib/spark-assembly-1.6.2-hadoop2.6.0-cdh5.4.7.jar org.apache.spark.launcher.Main org.apache.spark.deploy.SparkSubmit --master yarn-client --class SparkPi ./testJars/my.jar
```
将这个命令执行的结果返回给 CMD参数，然后执行

#### launcher.Main

这个类的目的是同时适应unix和windows操作系统

``` java org.apache.spark.launcher.Main.java
if (className.equals("org.apache.spark.deploy.SparkSubmit")) {
      try {
      // 创建一个命令解析器，这里会优先将提交的命令中的 --master之类的参数解析，然后保存到 SparkSubmitCommandBuilder 中
        builder = new SparkSubmitCommandBuilder(args);
      } catch (IllegalArgumentException e) {
        ...
      }
    } else {
      builder = new SparkClassCommandBuilder(className, args);
    }
    Map<String, String> env = new HashMap<String, String>();
    // 使用解析器来解析参数，并且得到环境变量的HashMap值
    List<String> cmd = builder.buildCommand(env);
    ...
    if (isWindows()) {
      System.out.println(prepareWindowsCommand(cmd, env));
    } else {
      // In bash, use NULL as the arg separator since it cannot be used in an argument.
      // 根据输入的参数，准备bash的命令，然后打印出来，传给 spark-class脚本中的$CMD 变量，然后执行，
      List<String> bashCmd = prepareBashCommand(cmd, env);
      for (String c : bashCmd) {
        System.out.print(c);
        System.out.print('\0');
      }
    }
```
再来看看这个cmd到底是怎么build的：
``` java org.apache.spark.launcher.SparkSubmitCommandBuilder.java
  // 这段代码进入 buildSparkSubmitCommand
  @Override
  public List<String> buildCommand(Map<String, String> env) throws IOException {
    if (PYSPARK_SHELL_RESOURCE.equals(appResource) && !printInfo) {
      return buildPySparkShellCommand(env);
    } else if (SPARKR_SHELL_RESOURCE.equals(appResource) && !printInfo) {
      return buildSparkRCommand(env);
    } else {
      return buildSparkSubmitCommand(env);
    }
  }
 ```
 看看buildSparkSubmitCommand函数
 ``` java
  ...
   private List<String> buildSparkSubmitCommand(Map<String, String> env) throws IOException {
    // Load the properties file and check whether spark-submit will be running the app's driver
    // or just launching a cluster app. When running the driver, the JVM's argument will be
    // modified to cover the driver's configuration.
    // 先获取有效的配置，如果用户不自己制定的话，默认情况下会去 $SPARK_HOME/conf/spark-defaults.conf 中拿
    Map<String, String> config = getEffectiveConfig();
    boolean isClientMode = isClientMode(config);
    String extraClassPath = isClientMode ? config.get(SparkLauncher.DRIVER_EXTRA_CLASSPATH) : null;

    List<String> cmd = buildJavaCommand(extraClassPath);
    // Take Thrift Server as daemon
    if (isThriftServer(mainClass)) {
      addOptionString(cmd, System.getenv("SPARK_DAEMON_JAVA_OPTS"));
    }
    addOptionString(cmd, System.getenv("SPARK_SUBMIT_OPTS"));
    addOptionString(cmd, System.getenv("SPARK_JAVA_OPTS"));

    // 接下来的代码的意思是，很多没有手动指定的参数，会根据配置文件中拿，如果配置文件中没有，会给个默认值
    if (isClientMode) {
      // Figuring out where the memory value come from is a little tricky due to precedence.
      // Precedence is observed in the following order:
      // - explicit configuration (setConf()), which also covers --driver-memory cli argument.
      // - properties file.
      // - SPARK_DRIVER_MEMORY env variable
      // - SPARK_MEM env variable
      // - default value (1g)
      // Take Thrift Server as daemon
      String tsMemory =
        isThriftServer(mainClass) ? System.getenv("SPARK_DAEMON_MEMORY") : null;
      String memory = firstNonEmpty(tsMemory, config.get(SparkLauncher.DRIVER_MEMORY),
        System.getenv("SPARK_DRIVER_MEMORY"), System.getenv("SPARK_MEM"), DEFAULT_MEM);
      cmd.add("-Xms" + memory);
      cmd.add("-Xmx" + memory);
      addOptionString(cmd, config.get(SparkLauncher.DRIVER_EXTRA_JAVA_OPTIONS));
      mergeEnvPathList(env, getLibPathEnvName(),
        config.get(SparkLauncher.DRIVER_EXTRA_LIBRARY_PATH));
    }

    addPermGenSizeOpt(cmd);
    cmd.add("org.apache.spark.deploy.SparkSubmit");
    cmd.addAll(buildSparkSubmitArgs());
    return cmd;
  }
```
#### SparkSubmit类
通过上述方法生成了命令到 $CMD 参数中后，就通过 `exec "${CMD[@]}"` 命令执行之前生成的命令，也就是 `org.apache.spark.deploy.SparkSubmit` 类。大概是这样子：
``` sh
// 在Unix中，分隔符为'\0'，以下是大概写法
/bin/java -Xms1g -XX:MaxPermSize=256m -cp /usr/lib/spark/lib/spark-assembly-1.6.2-hadoop2.6.0-cdh5.4.7.jar org.apache.spark.deploy.SparkSubmit --master yarn-client --class SparkPi ./testJars/my.jar
```
进入main函数：
``` scala
def main(args: Array[String]): Unit = {
    val appArgs = new SparkSubmitArguments(args)
    if (appArgs.verbose) {
      // scalastyle:off println
      printStream.println(appArgs)
      // scalastyle:on println
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    }
  }
```
咱们调用的是 submit(appArgs)：如下代码，主要分为两步，第一步，准备提交的参数四元组，第二步，
``` scala
private def submit(args: SparkSubmitArguments): Unit = {
    // 这里的代码很长，主要目的是准备提交应用的环境，针对 yarn standalone Mesos等各类环境进行针对性处理，并对输入的args进行一些校验和修改，返回一个 4-tuple：
    // 1. 子程序的参数列表； 2. 子程序的classpath列表 3. 系统环境变量HashMap 4. mainClass
    val (childArgs, childClasspath, sysProps, childMainClass) = prepareSubmitEnvironment(args)

    // 如果有代理晕乎的话，需要创建一个代理用户，然后验证，后运行runMain
    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
            }
          })
        } catch {
          case e: Exception =>
            // Hadoop's AuthorizationException suppresses the exception's stack trace, which
            // makes the message printed to the output by the JVM not very helpful. Instead,
            // detect exceptions with empty stack traces here, and treat them differently.
            if (e.getStackTrace().length == 0) {
              // scalastyle:off println
              printStream.println(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
              // scalastyle:on println
              exitFn(1)
            } else {
              throw e
            }
        }
      } else {
        runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
      }
    }

     // In standalone cluster mode, there are two submission gateways:
     //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
     //   (2) The new REST-based gateway introduced in Spark 1.3
     // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
     // to use the legacy gateway if the master endpoint turns out to be not a REST server.
。。。
  }
```
接下来执行的是runMain方法：

``` scala
//复用反射加载childMainClass
//调用反射机制加载main方法
//执行main方法,进入 SparkPi 的main方法，执行spark应用程序
```
至此，正式完成spark应用程序的提交。


#### 引用
http://www.cnblogs.com/xing901022/p/6426408.html
http://blog.csdn.net/lovehuangjiaju/article/details/49123975