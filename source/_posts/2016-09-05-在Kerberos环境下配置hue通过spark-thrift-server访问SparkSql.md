---
title: 在Kerberos环境下配置hue通过spark-thrift-server访问SparkSql
date: 2016-09-05 13:47:02
toc: true
comments: true
tags: 
- spark
- hue
- kerberos
categories:
- spark
---
hue-spark-thriftserver-kerberos
### 背景说明
Kerberos项目最后要对基于Hue的TODP平台进行安全测试，在搭建配置的过程中踩了一些坑，现在把其中的配置与步骤进行总结，以免以后忘记。

其中用到以下代号：
40机器：hue平台所在的机器
76机器：spark thrift服务端口10010，hive-thrift-server服务端口10000
74机器：spark thrift服务端口10010，hive-thrift-server服务端口10000
TEST-BDD-HIVESERVER机器：负载均衡所在的机器，负载均衡机器需要配合开启10000和10010端口

在kerberos认证下, sparksql的thriftserver连接hiveserver2变得相对复杂，主要是因为各种kerberos认证出现各种问题。后来由于hive使用了负载均衡，所以spark-sql也需加入负载均衡，否则不能使用，就是这个负载均衡服务器的加入使得kerberos认证变得更加复杂，使得不明原理的新手在配置kerberos的keytab与principal时各种不匹配。这里是通过Hue可视化界面调用后台的sparksql,然后sparksql通过JDBC连接Hive的hiveServer2服务。

### 40机器hue端配置

进入40机器hue所在的目录
``` bash hue.ini
$ cd /usr/lib/hue/ 
$ vim desktop/conf/hue.ini
```
修改hue的配置文件如下
``` bash
1119 [spark]
...
1134   # spark-sql config
1135   spark_sql_server_host=TEST-BDD-HIVESERVER
1136   ## spark_sql_server_port=10010
```
由于此处使用了负载均衡，所以上述TEST-BDD-HIVESERVER指向的是负载均衡所在的ip，最终会转发给两个spark-thrift-server

### Kerberos服务器端配置
生成类似 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN 的keytab，配置了负载均衡后，使用test-bdd-hiveserver

### 76机器上的配置
76机器与74机器配置步骤一样，只是hive-site.xml需要改一处，将下面的 076改成 074即可
``` bash hive-site.xml
<property>
   <name>hive.server2.thrift.bind.host</name>
   <value>test-bdd-076</value>
   <description>TCP port number to listen on, default 10000</description>
</property>
```

其它都一样，所以在这里只写076的配置步骤

#### 从hive.keytab创建spark的keytab
然后在/etc/hive/conf/下创建spark需要的keytab，在这里使用hiveserver的keytab，将已有的hive.keytab_hiveserver 拷贝成 hive.keytab_sparkthrift，然后修改权限如下：

``` bash
-rw------- 1 hive hive     424 8月  23 09:55 hive.keytab_hiveserver
-rw------- 1 op   bigdata  424 9月   3 12:25 hive.keytab_sparkthrift
```
修改好后用如下命令检查：

``` bash
$ sudo klist -k hive.keytab_sparkthrift 
Keytab name: FILE:hive.keytab_sparkthrift
KVNO Principal
---- --------------------------------------------------------------------------
   1 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN
   1 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN
   1 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN
   1 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN
   1 hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN
```
如果klist是如上结果，就对了

#### 配置spark需要的hive-site.xml

由于需要修改hive的一些配置，进入76机器spark所在的目录，将`/etc/hive/conf/`下的`hive-site.xml`拷贝到spark的conf下，赋予权限并修改
``` bash
$ sudo cp /etc/hive/conf/hive-site.xml $SPARK_HOME/conf/
$ cd $SPARK_HOME
$ sudo chmod op conf/hive-site.xml
$ vim conf/hive-site.xml
```
修改hive-site.xml,增加hive.server2.thrift.bind.host

``` bash hive-site.xml
<!-- ZooKeeper conf-->
<property>
  <name>hive.server2.enable.doAs</name>
  <value>false</value>
  <description> Impersonate the connected user </description>
</property>
<property>
  <name>hive.server2.thrift.port</name>
  <value>10010</value>
  <description>TCP port number to listen on, default 10000</description>
</property>

<property>
   <name>hive.server2.thrift.bind.host</name>
   <value>test-bdd-076</value>
   <description>TCP port number to listen on, default 10000</description>
 </property>

<property>
  <name>hive.metastore.execute.setugi</name>
  <value>true</value>
</property>
<property>
   <name>hive.server2.authentication.kerberos.principal</name>
   <value>hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN</value>
 </property>
<property>
  <name>hive.server2.authentication.kerberos.keytab</name>
  <value>/etc/hive/conf/hive.keytab_sparkthrift</value>
</property>

#### 启动Spark-thrift-server
``` bash
$ cd $SPARK_HOME
$ ./sbin/start-thriftserver.sh
```
可以通过如下日志查看是否启动成功：
``` bash
$ vim logs/spark-op-org.apache.spark.sql.hive.thriftserver.HiveThriftServer2-1-TEST-BDD-076.out 
```
启动成功会看到如下日志:
``` bash
  96 16/09/05 13:41:25 INFO AbstractService: Service:HiveServer2 is started.
  97 16/09/05 13:41:25 INFO HiveThriftServer2: HiveThriftServer2 started
  98 16/09/05 13:41:25 INFO UserGroupInformation: Login successful for user hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN using keytab file /etc/hive/conf/hive.keytab_sparkthrift
  99 16/09/05 13:41:25 INFO AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens
 100 16/09/05 13:41:25 INFO TokenStoreDelegationTokenSecretManager: New master key with key id=0
 101 16/09/05 13:41:25 INFO TokenStoreDelegationTokenSecretManager: Starting expired delegation token remover thread, tokenRemoverScanInterval=60 min(s)
 102 16/09/05 13:41:25 INFO AbstractDelegationTokenSecretManager: Updating the current master key for generating delegation tokens
 103 16/09/05 13:41:25 INFO TokenStoreDelegationTokenSecretManager: New master key with key id=1
 104 16/09/05 13:41:25 INFO ThriftCLIService: Starting ThriftBinaryCLIService on port 10010 with 5...500 worker threads
 ```

### 负载均衡机器的查看
进入 67.121机器
输入 命令 `sudo ipvsadm -ln`

``` bash
$ sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4194304)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.142.67.123:10000 wlc persistent 7200 synproxy
  -> 10.142.78.74:10000           FullNat 50     3          0         
  -> 10.142.78.76:10000           FullNat 50     0          0         
TCP  10.142.67.123:10010 wlc persistent 7200 synproxy
  -> 10.142.78.74:10010           FullNat 50     0          0         
  -> 10.142.78.76:10010           FullNat 50     2          0    
```
就可以看到负载均衡的情况了：

### 踩坑说明以及解决方案

#### 缺少配置kerberos认证错误
需要在hive-site.xml文件中添加kerberos认证相关配置
#### kerberos认证失败
1)  在hive-site.xml中配置好kerberos认证，但是op用户下无法读取hive.keytab的问题，出现unable to login ...given keytab/principal 以及Unable to obtain password from user。因为hive.keytab 是hive用户创建的，op用户无法读取，导致看似kerberos已经配置好，
但是程序没有读取权限，依旧认为没有配置好，这是会有在日志文件中会有NULLPOINT类似的错误提示，说明是没有读取权限。解决方案是复制hive.keytab到op用户下。
2）在hue界面连接spark时可能会出现10010端口不能连接的问题，这是sparkthrift没有启动导致的；
3）spark thriftserver明明已经启动，但是hue界面仍旧不能连接，出现TTransportException的错误，原因是kerberos配置没有配置正确，即没有配置kerberos认证的keytab与principal。hive/test-bdd-hiveserver必须与hive.keytab_hiveserver配套使用，同理，test-bdd-074或者 test-bdd-076必须与hive/test-bdd-74或者hive/test-bdd-76配套使用，否则出现认证失败的问题。
#### hue的配置问题。
在hue的desktop/conf目录下hue.ini文件中，主要配置spark_sql_server_host，也就是spark thriftserver所在主机，这里可以是负载均衡服务器TEST-BDD-HIVESERVER,spark_sql_server_port 是spark thriftserver的服务端口。
需要注意的是，加上kerberos认证后，主机名不能是ip地址的形式，需要FQDN的形式。hive的配置需要注意的是hive_server_host，这里绝对不能是hiveserver2的服务器的地址，一定是负载均衡服务器的地址，不然在hue界面连接HIVE时出现
Unable to access databases, Query Server or Metastore may be down.的错误以及GSS initial failed的错误，无法访问hive数据库。
#### metastore的问题
连接metastore也需要principal的认证。
``` bash
20 <property>
221   <name>hive.metastore.sasl.enabled</name>
222   <value>true</value>
223   <description>If true, the metastore thrift interface will be secured with SASL. Clients must authenticate with Kerberos.</description>
224 </property>
225 <property>
226   <name>hive.metastore.kerberos.principal</name>
227   <value>hive/_HOST@HADOOP.CHINATELECOM.CN</value>
228   <description>The service principal for the metastore thrift server. The special string _HOST will be replaced automatically with the correct host name.</description>
229 </property>
```

之所以问题多多，主要原因是对kerberos+Hive+lvs整体原理没有搞清楚，以至于在配置过程中出现各种错误。我们搭建的hive集群有74,76两台主机，spark thriftserver也有74,76两台主机，负载均衡服务器在test-bdd-hiveserver上。在配置时，需要将spark-sql-server-host配置成test-bdd-hiveserver,因为对spark而言，74与76上的hiveserver是一个整体，不能配置成单一的主机，不然lvs可能会将服务分到另外一台主机上，造成主机配置失败。