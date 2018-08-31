---
title: kerberos下spark客户端的配置
date: 2016-09-06 09:43:27
toc: true
tags: 
- spark
- kerberos
categories:
- spark
---

Kerberos环境下spark的客户端配置并不是很多，主要需要配置的是spark-history与spark-sql

软件版本：spark-1.6.2

注：正式环境中，需要将spark客户端的路径放入其它短路经，比如 /etc/local/spark 等
``` bash spark-env.sh
# 由于此处的 hive-site.xml 需要做一定修改，所以需要将hive-site.xml core-site.xml hdfs-site.xml yarn-site.xml等导入conf文件夹下
export HADOOP_CONF_DIR=/usr/op/sparkKerbersTest/spark-1.6.2-bin-hadoop2.6/conf

export JAVA_HOME=/usr/java/jdk1.7.0_75
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/hadoop/lib/native/
export SPARK_LIBRARY_PATH=/usr/lib/hadoop/lib/native/:$SPARK_LIBRARY_PATH

# history需要的Kerberos配置
SPARK_HISTORY_OPTS="-Dspark.history.ui.port=8777 -Dspark.history.retainedApplications=10 -Dspark.history.fs.logDirectory=hdfs://ns/user/op/sparkHistoryServer -Dspark.history.kerberos.enabled=true -Dspark.history.kerberos.principal=op    @HADOOP.CHINATELECOM.CN -Dspark.history.kerberos.keytab=/usr/op/sparkKerbersTest/spark-1.6.2-bin-hadoop2.6/conf/op.keytab"
```

#### 从hive.keytab_hiveserver创建spark-thrift-server的keytab
``` bash
-rw------- 1 hive hive     424 8月  23 09:55 hive.keytab_hiveserver
-rw------- 1 op   bigdata  424 9月   3 12:25 hive.keytab_sparkthrift
```
#### hive-site的配置

修改hive-site.xml：
- 增加hive.server2.thrift.bind.host
- 修改hive.server2.thrift.port为10010
- 修改hive.server2.authentication.kerberos.keytab为如下

``` bash hive-site.xml
145 <!-- ZooKeeper conf-->
146 <property>
147   <name>hive.server2.enable.doAs</name>
148   <value>false</value>
149   <description> Impersonate the connected user </description>
150 </property>
151 <property>
152   <name>hive.server2.thrift.port</name>
153   <value>10010</value>
154   <description>TCP port number to listen on, default 10000</description>
155 </property>
156 
157 <property>
158    <name>hive.server2.thrift.bind.host</name>
159    <value>test-bdd-076</value>
160    <description>TCP port number to listen on, default 10000</description>
161  </property>
162 
163 <property>
164   <name>hive.metastore.execute.setugi</name>
165   <value>true</value>
166 </property>
...
209 <property>
210    <name>hive.server2.authentication.kerberos.principal</name>
211    <value>hive/test-bdd-hiveserver@HADOOP.CHINATELECOM.CN</value>
212  </property>
213 <property>
214   <name>hive.server2.authentication.kerberos.keytab</name>
215   <value>/etc/hive/conf/hive.keytab_sparkthrift</value>
216 </property>
```
