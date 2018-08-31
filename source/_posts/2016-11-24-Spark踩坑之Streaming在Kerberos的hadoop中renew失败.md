---
title: Spark踩坑之Streaming在Kerberos的hadoop中renew失败
toc: true
date: 2016-11-24 16:24:50
tags: 
- spark streaming
- kerberos
- spark开发
categories: spark开发
---

## 问题描述：
SparkStreaming任务的Kerberos环境下两天后出现 AMRMTOKEN INVALID

早上回来，跑在yarn上面的Streaming程序莫名奇妙崩溃了，yarn logs 查看日志，700万行，发现在每个executor的最后报错如下：
``` bash
16/11/01 15:02:05 INFO executor.CoarseGrainedExecutorBackend: Registered signal handlers for [TERM, HUP, INT]
16/11/01 15:02:06 INFO spark.SecurityManager: Changing view acls to: wzfw
16/11/01 15:02:06 INFO spark.SecurityManager: Changing modify acls to: wzfw
16/11/01 15:02:06 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(wzfw); users with modify permissions: Set(wzfw)
...
16/11/02 03:51:33 ERROR executor.CoarseGrainedExecutorBackend: RECEIVED SIGNAL 15: SIGTERM
...
16/11/02 03:51:34 ERROR executor.CoarseGrainedExecutorBackend: RECEIVED SIGNAL 15: SIGTERM
16/11/02 03:51:34 WARN executor.CoarseGrainedExecutorBackend: An unknown (NM-304-SA5212M4-BIGDATA-519:44457) driver disconnected.
16/11/02 03:51:34 ERROR executor.CoarseGrainedExecutorBackend: Driver 10.142.116.19:44457 disassociated! Shutting down.
```

这个报错应该只是表象，原因应该出自driver，定位到driver： NM-304-SA5212M4-BIGDATA-519
``` bash
16/11/02 03:51:34 WARN executor.CoarseGrainedExecutorBackend: An unknown (NM-304-SA5212M4-BIGDATA-519:44457) driver disconnected.
16/11/02 03:51:34 ERROR executor.CoarseGrainedExecutorBackend: Driver 10.142.116.19:44457 disassociated! Shutting down.
16/11/02 03:51:34 ERROR executor.CoarseGrainedExecutorBackend: RECEIVED SIGNAL 15: SIGTERM
16/11/02 03:51:34 ERROR client.TransportClient: Failed to send RPC 5285936577870185909 to NM-304-SA5212M4-BIGDATA-519/10.142.116.19:44457: java.nio.channels.ClosedChannelException
java.nio.channels.ClosedChannelException
16/11/02 03:51:34 WARN netty.NettyRpcEndpointRef: Error sending message [message = Heartbeat(53,[Lscala.Tuple2;@519f65d9,BlockManagerId(53, NM-304-SA5212M4-BIGDATA-382, 29163))] in 1 attempts
java.io.IOException: Failed to send RPC 5285936577870185909 to NM-304-SA5212M4-BIGDATA-519/10.142.116.19:44457: java.nio.channels.ClosedChannelException
        at org.apache.spark.network.client.TransportClient$3.operationComplete(TransportClient.java:239)
        at org.apache.spark.network.client.TransportClient$3.operationComplete(TransportClient.java:226)
        at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:680)
        at io.netty.util.concurrent.DefaultPromise.notifyListeners(DefaultPromise.java:567)
        at io.netty.util.concurrent.DefaultPromise.tryFailure(DefaultPromise.java:424)
        at io.netty.channel.AbstractChannel$AbstractUnsafe.safeSetFailure(AbstractChannel.java:801)
        at io.netty.channel.AbstractChannel$AbstractUnsafe.write(AbstractChannel.java:699)
        at io.netty.channel.DefaultChannelPipeline$HeadContext.write(DefaultChannelPipeline.java:1122)
        at io.netty.channel.AbstractChannelHandlerContext.invokeWrite(AbstractChannelHandlerContext.java:633)
        at io.netty.channel.AbstractChannelHandlerContext.access$1900(AbstractChannelHandlerContext.java:32)
        at io.netty.channel.AbstractChannelHandlerContext$AbstractWriteTask.write(AbstractChannelHandlerContext.java:908)
        at io.netty.channel.AbstractChannelHandlerContext$WriteAndFlushTask.write(AbstractChannelHandlerContext.java:960)
        at io.netty.channel.AbstractChannelHandlerContext$AbstractWriteTask.run(AbstractChannelHandlerContext.java:893)
        at io.netty.util.concurrent.SingleThreadEventExecutor.runAllTasks(SingleThreadEventExecutor.java:357)
        at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:357)
        at io.netty.util.concurrent.SingleThreadEventExecutor$2.run(SingleThreadEventExecutor.java:111)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.nio.channels.ClosedChannelException
```
找到driver的日志：
``` bash
Container: container_1477044851292_468337_02_000004 on NM-304-SA5212M4-BIGDATA-519_8041
=========================================================================================
LogType:stderr
Log Upload Time:星期三 十一月 02 03:51:47 +0800 2016
LogLength:194115722
Log Contents:
```
根据时间定位：（vim中在driver中搜索 16\/11\/02 03:51:3）

发现问题出题的源头：
在 16/11/02 03:40 之前，driver的日志还是比较稳定的，但是在 此之后，频繁出现下述异常：

``` bash
16/11/01 15:11:43 INFO ApplicationMaster: Registered signal handlers for [TERM, HUP, INT]
16/11/01 15:11:43 INFO ApplicationMaster: ApplicationAttemptId: appattempt_1477044851292_468337_000002
16/11/01 15:11:45 INFO SecurityManager: Changing view acls to: wzfw
16/11/01 15:11:45 INFO SecurityManager: Changing modify acls to: wzfw
16/11/01 15:11:45 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(wzfw); users with modify permissions: Set(wzfw)
...
16/11/02 03:41:17 WARN Client: Exception encountered while connecting to the server : org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.token.SecretManager$InvalidToken): Invalid AMRMToken from appattempt_1477044851292_468337_000002
16/11/02 03:41:17 INFO RetryInvocationHandler: Exception while invoking allocate of class ApplicationMasterProtocolPBClientImpl over rm1. Trying to fail over immediately.
org.apache.hadoop.security.token.SecretManager$InvalidToken: Invalid AMRMToken from appattempt_1477044851292_468337_000002
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:57)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
        at org.apache.hadoop.yarn.ipc.RPCUtil.instantiateException(RPCUtil.java:53)
        at org.apache.hadoop.yarn.ipc.RPCUtil.unwrapAndThrowException(RPCUtil.java:104)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:79)
        at sun.reflect.GeneratedMethodAccessor137.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:187)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
        at com.sun.proxy.$Proxy19.allocate(Unknown Source)
        at org.apache.hadoop.yarn.client.api.impl.AMRMClientImpl.allocate(AMRMClientImpl.java:278)
        at org.apache.spark.deploy.yarn.YarnAllocator.allocateResources(YarnAllocator.scala:225)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$1.run(ApplicationMaster.scala:384)
Caused by: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.token.SecretManager$InvalidToken): Invalid AMRMToken from appattempt_1477044851292_468337_000002
        at org.apache.hadoop.ipc.Client.call(Client.java:1468)
        at org.apache.hadoop.ipc.Client.call(Client.java:1399)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:232)
        at com.sun.proxy.$Proxy18.allocate(Unknown Source)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:77)
        ... 9 more
16/11/02 03:41:17 INFO ConfiguredRMFailoverProxyProvider: Failing over to rm2
16/11/02 03:41:17 INFO RetryInvocationHandler: Exception while invoking allocate of class ApplicationMasterProtocolPBClientImpl over rm2 after 1 fail over attempts. Trying to fail over after sleeping for 22672ms.
java.net.ConnectException: Call From NM-304-SA5212M4-BIGDATA-519/10.142.116.19 to NM-304-RH5885V3-BIGDATA-008:8030 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:57)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
        at org.apache.hadoop.net.NetUtils.wrapWithMessage(NetUtils.java:791)
        at org.apache.hadoop.net.NetUtils.wrapException(NetUtils.java:731)
        at org.apache.hadoop.ipc.Client.call(Client.java:1472)
        at org.apache.hadoop.ipc.Client.call(Client.java:1399)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:232)
        at com.sun.proxy.$Proxy18.allocate(Unknown Source)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:77)
        at sun.reflect.GeneratedMethodAccessor137.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:187)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
        at com.sun.proxy.$Proxy19.allocate(Unknown Source)
        at org.apache.hadoop.yarn.client.api.impl.AMRMClientImpl.allocate(AMRMClientImpl.java:278)
        at org.apache.spark.deploy.yarn.YarnAllocator.allocateResources(YarnAllocator.scala:225)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$1.run(ApplicationMaster.scala:384)
Caused by: java.net.ConnectException: Connection refused
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:739)
        at org.apache.hadoop.net.SocketIOWithTimeout.connect(SocketIOWithTimeout.java:206)
        at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:530)
        at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:494)
        at org.apache.hadoop.ipc.Client$Connection.setupConnection(Client.java:607)
        at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:705)
        at org.apache.hadoop.ipc.Client$Connection.access$2800(Client.java:368)
        at org.apache.hadoop.ipc.Client.getConnection(Client.java:1521)
        at org.apache.hadoop.ipc.Client.call(Client.java:1438)
        ... 13 more
16/11/02 03:41:20 INFO JobScheduler: Added jobs for time 1478029280000 ms
```

然后这些日志持续出现了十分钟左右，当然spark的stage是依旧在增加，依旧在运行

``` bash
16/11/02 03:51:27 INFO RetryInvocationHandler: Exception while invoking allocate of class ApplicationMasterProtocolPBClientImpl over rm1 after 14 fail over attempts. Trying to fail over immediately.
org.apache.hadoop.security.token.SecretManager$InvalidToken: Invalid AMRMToken from appattempt_1477044851292_468337_000002
        at sun.reflect.GeneratedConstructorAccessor74.newInstance(Unknown Source)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
        at org.apache.hadoop.yarn.ipc.RPCUtil.instantiateException(RPCUtil.java:53)
        at org.apache.hadoop.yarn.ipc.RPCUtil.unwrapAndThrowException(RPCUtil.java:104)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:79)
        at sun.reflect.GeneratedMethodAccessor137.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:187)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
        at com.sun.proxy.$Proxy19.allocate(Unknown Source)
        at org.apache.hadoop.yarn.client.api.impl.AMRMClientImpl.allocate(AMRMClientImpl.java:278)
        at org.apache.spark.deploy.yarn.YarnAllocator.allocateResources(YarnAllocator.scala:225)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$1.run(ApplicationMaster.scala:384)
Caused by: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.token.SecretManager$InvalidToken): Invalid AMRMToken from appattempt_1477044851292_468337_000002
        at org.apache.hadoop.ipc.Client.call(Client.java:1468)
        at org.apache.hadoop.ipc.Client.call(Client.java:1399)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:232)
        at com.sun.proxy.$Proxy18.allocate(Unknown Source)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:77)
        ... 9 more
16/11/02 03:51:27 INFO ConfiguredRMFailoverProxyProvider: Failing over to rm2
16/11/02 03:51:27 INFO RetryInvocationHandler: Exception while invoking allocate of class ApplicationMasterProtocolPBClientImpl over rm2 after 15 fail over attempts. Trying to fail over after sleeping for 31772ms.
java.net.ConnectException: Call From NM-304-SA5212M4-BIGDATA-519/10.142.116.19 to NM-304-RH5885V3-BIGDATA-008:8030 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused
        at sun.reflect.GeneratedConstructorAccessor75.newInstance(Unknown Source)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
        at org.apache.hadoop.net.NetUtils.wrapWithMessage(NetUtils.java:791)
        at org.apache.hadoop.net.NetUtils.wrapException(NetUtils.java:731)
        at org.apache.hadoop.ipc.Client.call(Client.java:1472)
        at org.apache.hadoop.ipc.Client.call(Client.java:1399)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:232)
        at com.sun.proxy.$Proxy18.allocate(Unknown Source)
        at org.apache.hadoop.yarn.api.impl.pb.client.ApplicationMasterProtocolPBClientImpl.allocate(ApplicationMasterProtocolPBClientImpl.java:77)
        at sun.reflect.GeneratedMethodAccessor137.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:187)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
        at com.sun.proxy.$Proxy19.allocate(Unknown Source)
        at org.apache.hadoop.yarn.client.api.impl.AMRMClientImpl.allocate(AMRMClientImpl.java:278)
        at org.apache.spark.deploy.yarn.YarnAllocator.allocateResources(YarnAllocator.scala:225)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$1.run(ApplicationMaster.scala:384)
Caused by: java.net.ConnectException: Connection refused
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:739)
        at org.apache.hadoop.net.SocketIOWithTimeout.connect(SocketIOWithTimeout.java:206)
        at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:530)
        at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:494)
        at org.apache.hadoop.ipc.Client$Connection.setupConnection(Client.java:607)
        at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:705)
        at org.apache.hadoop.ipc.Client$Connection.access$2800(Client.java:368)
        at org.apache.hadoop.ipc.Client.getConnection(Client.java:1521)
        at org.apache.hadoop.ipc.Client.call(Client.java:1438)
        ... 13 more
```

然后在最后收到这个信号就挂了：

``` bash
16/11/02 03:51:31 INFO WriteAheadLogManager  for Thread: Attempting to clear 0 old log files in hdfs://ns/user/wzfw/checkpoint/receivedBlockMetadata older than 1478029880000:
16/11/02 03:51:31 INFO InputInfoTracker: remove old batch metadata: 1478029870000 ms
16/11/02 03:51:32 INFO ApplicationMaster: Final app status: FAILED, exitCode: 16
16/11/02 03:51:32 WARN ApplicationMaster: Reporter thread fails 2 time(s) in a row.
java.lang.reflect.UndeclaredThrowableException
        at com.sun.proxy.$Proxy19.allocate(Unknown Source)
        at org.apache.hadoop.yarn.client.api.impl.AMRMClientImpl.allocate(AMRMClientImpl.java:278)
        at org.apache.spark.deploy.yarn.YarnAllocator.allocateResources(YarnAllocator.scala:225)
        at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$1.run(ApplicationMaster.scala:384)
Caused by: java.lang.InterruptedException: sleep interrupted
        at java.lang.Thread.sleep(Native Method)
        at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:151)
        ... 4 more
16/11/02 03:51:32 ERROR ApplicationMaster: RECEIVED SIGNAL 15: SIGTERM
16/11/02 03:51:33 INFO StreamingContext: Invoking stop(stopGracefully=false) from shutdown hook
16/11/02 03:51:33 INFO JobGenerator: Stopping JobGenerator immediately
```
后来又跑了一次，又因为同样的原因挂了，这次统计了时间：
slaver：
程序启动时间：16/11/02 22:58:27
程序结束时间： 16/11/03 22:51:34

driver 
程序启动时间：16/11/02 22:58:18
遇到Kerberos问题时间： 16/11/03 22:41:23
程序结束时间：16/11/03 22:51:35

上面的时间是有不正确的地方的，因为我启动的时间肯定不是16/11/02 22:58:18（那个时候我已经回家了），我是在 11/02号当天上午启动，所以一个很可能的原因是，在22:58的这个时候，因为种种原因，程序崩溃重启，换了一个driver。

## 可能的原因

google一下，网上查到有类似的原因：
storm on yarn在这个问题中存在一些bug；
https://community.cloudera.com/t5/Batch-Processing-and-Workflow/Long-running-yarn-app-storm-yarn-exits-with-Invalid-AMRMToken/td-p/26717

http://www.codeflitting.com/blog/article/%E4%B8%BA%E9%95%BF%E6%97%B6%E9%97%B4%E8%BF%90%E8%A1%8C%E7%9A%84%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E9%85%8D%E7%BD%AE%20YARN

https://issues.apache.org/jira/browse/YARN-3103

根据日志以及网上资料，初步认为与Kerberos过期认证有关，AMRMToken无效，也就是说，在运行了一段时期（一天多）后，AM到RM的token无效了，然后无效很多次后，后续的任务分配不到container，所以程序就挂了，突然想到，出现这种错误的原因有以下几种；

1. 大约12个小时以前，SparkStreaming程序因为后面的flume测试守护进程，导致换了一个driver，是否是由于driver改动后，导致token失效？

2. yarn 2.6.0有一个相关的bug，https://issues.apache.org/jira/browse/YARN-3103 
AMRMClientImpl.updateAMRMToken updates the token service before storing it to the credentials, so the token is mapped using the newly updated service rather than the empty service that was used when the RM created the original AMRM token. This leads to two AMRM tokens in the credentials and can still fail if the AMRMTokenSelector picks the wrong one.
In addition the AMRMClientImpl grabs the login user rather than the current user when security is enabled, so it's likely the UGI being updated is not the UGI that will be used when reconnecting to the RM.
The end result is that AMs can fail with invalid token errors when trying to reconnect to an RM after a new AMRM secret has been activated.
大概意思是，更新token的时候会有两个token，如果选择了错误的token，就会导致出错

3. hadoop设置仅允许 hdfs 用户的委派令牌保留最大生存期 7 （default）天，这始终是不够的
需要修改一下yarn的配置，增加如下参数：

将 ResourceManager 配置为对应 HDFS NameNode 的代理用户，以便在现有令牌超过其最大生存期时，ResourceManager 可以请求新的令牌。YARN 随后能够代表 hdfs 用户继续执行本地化和日志聚合

``` xml yarn-site.xml
<property>
    <name>yarn.resourcemanager.proxy-user-privileges.enabled</name>
    <value>true</value>
</property>


``` xml core-site.xml
<property>
    <name>hadoop.proxyuser.yarn.hosts</name>
    <value>*</value>
</property>

<property>
    <name>hadoop.proxyuser.yarn.groups</name>
    <value>*</value>
</property>
```
然后重启YARN和HDFS服务

## 问题的暂时解决
由于项目紧张，我写了一个脚本，每隔一分钟监控yarn上面运行的spark Streaming程序，如果挂了就重新启动起来，虽然也不影响后续的结果。后来观察，基本上是一天一挂或者两天一挂，不过这也太不靠谱了。

## 问题解决
后来发现都不是上述原因，同事的前爱奇艺的同事遇到过同样的问题：原因是：
** NameNode采用了HA后，AM与Namenode通信使用的token的结构变为HA token,HA token中会有两个private token，代表两台namenode服务。 当AM更新token时，会调用hadoop客户端的addDelegationTokens更新token。但addDelegationTokens存在问题，其只会更新HA token，不会更新private token，而AM向NameNode发起请求时，会使用private token，导致出现异常。**

发生这个问题需要以下几个条件：
``` bash
1. NameNode HA is enabled.
2. Kerberos is enabled.
3. HDFS Delegation Token (not Keytab or TGT) is used to communicate with NameNode.
4. We want to update the HDFS Delegation Token for long running applicatons. HDFS Client will generate private tokens for each NameNode. When we update the HDFS Delegation Token, these private tokens will not be updated, which will cause token expired.
```
### hadoop修复方案
这是hadoop的一个bug，在hadoop 2.9的新版本中修复了这个问题，所以，给hadoop打个patch就好了：

Hadoop: https://issues.apache.org/jira/browse/HDFS-9276 
hadoop2.9修复了private token不更新问题

### spark修复方案

Spark： https://github.com/apache/spark/pull/9168/files
主动使FileSystem对象关闭，再重新建立。FileSystem对象建立期间会重新设置private token

由于hadoop变动对生产环境的影响很大，所以我们选择了spark的修复方案，修改了源码后打包编译上线，问题解决；

感谢聪哥，也感谢那位在爱奇艺的发现并解决了这个问题的大神，没有你们，我要惨了。

与Kerberos对干的日子很酸爽

### 后记
还有一个方案可以解决这个问题，可以不需要修改spark源码，提交的时候增加参数：
```
--conf spark.hadoop.fs.hdfs.impl.disable.cache=true
```


#### 引用
http://mkuthan.github.io/blog/2016/09/30/spark-streaming-on-yarn/