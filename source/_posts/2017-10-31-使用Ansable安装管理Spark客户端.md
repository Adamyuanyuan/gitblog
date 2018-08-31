---
title: 使用Ansable安装管理Spark客户端
toc: true
date: 2017-10-31 20:52:03
tags: spark
categories: spark
---

随着生产的spark升级到2.2.0，我要为很多用户同时提供spark2.2.0以及spark1.6.2的客户端，这是一个比较繁琐耗时的操作，很烦，况且其它开发项目比较紧张，我决定学习使用Ansable来统一管理它们。

## 关于Ansable
略 配置管理的思想，基于推送的模式

## 安装spark客户端的思路
``` scala
for (eachNode <- sparkNodes: 所有客户端的机器) {
	1. 安装jdk1.8
	2. 从主控机传输已经打包好的spark客户端
	3. 解压到 /usr/local/spark2.2
	4. 配置环境变量
	5. (可选)配置history-server等其它客户端
	6. 运行自动化测试程序测试，并生成测试报告
}
```


tele-spark-version-change 1.6|2.2
通过该命令实现spark版本切换，核心就是 export相应的环境变量和PATH

## 使用方式

0. 首先，得打包编译更新好spark客户端，该配的东西配上，该改的jar包改掉，然后放到主控机的files目录下，修改ansible脚本的一些参数；
1. ssh到 102.133 （master机器，确保该机器与每台机器的ssh免密打通），cd到 spark_client_deploy.yml 所在目录
2. 确定想要修改的机器集群名称，修改 hosts文件，比如，要增加新机器组的安装，则增加如下配置：
``` sh
[yaxin2]
10.257.101.128
10.257.101.131
10.257.101.132
```
3. 执行脚本即可：
``` sh
ansible-playbook spark_client_deploy.yml 
```
如果没有啥问题，就会发现如下输出：

```
PLAY [deploy spark client(for version 1.6.2 or 2.2.1)] ************************ 

TASK: [ping] ****************************************************************** 
ok: [10.142.101.132]
ok: [10.142.101.131]
ok: [10.142.101.128]
ok: [10.142.101.129]
ok: [10.142.97.124]
ok: [10.142.101.130]

TASK: [mkdir] ***************************************************************** 
ok: [10.142.101.129]
ok: [10.142.101.131]
ok: [10.142.101.132]
ok: [10.142.101.128]
ok: [10.142.97.124]
ok: [10.142.101.130]

TASK: [test java8 is exist] *************************************************** 
changed: [10.142.101.129]
changed: [10.142.101.130]
changed: [10.142.101.132]
changed: [10.142.101.128]
changed: [10.142.101.131]
changed: [10.142.97.124]

TASK: [copy java8] ************************************************************ 
skipping: [10.142.101.128]
skipping: [10.142.97.124]
skipping: [10.142.101.129]
skipping: [10.142.101.130]
skipping: [10.142.101.131]
skipping: [10.142.101.132]

TASK: [extract java] ********************************************************** 
skipping: [10.142.101.129]
skipping: [10.142.97.124]
skipping: [10.142.101.128]
skipping: [10.142.101.132]
skipping: [10.142.101.130]
skipping: [10.142.101.131]

TASK: [copy ctcc-spark] ******************************************************* 
ok: [10.142.101.131]
ok: [10.142.101.128]
ok: [10.142.101.129]
ok: [10.142.101.132]
ok: [10.142.97.124]
changed: [10.142.101.130]

TASK: [rm old spark2] ********************************************************* 
changed: [10.142.101.130]
ok: [10.142.101.128]
ok: [10.142.101.131]
ok: [10.142.97.124]
ok: [10.142.101.132]
ok: [10.142.101.129]

TASK: [extract ctcc-spark] **************************************************** 
changed: [10.142.101.130]
changed: [10.142.101.131]
changed: [10.142.101.129]
changed: [10.142.101.132]
changed: [10.142.101.128]
changed: [10.142.97.124]

PLAY RECAP ******************************************************************** 
10.142.101.128             : ok=6    changed=2    unreachable=0    failed=0   
10.142.101.129             : ok=6    changed=2    unreachable=0    failed=0   
10.142.101.130             : ok=6    changed=4    unreachable=0    failed=0   
10.142.101.131             : ok=6    changed=2    unreachable=0    failed=0   
10.142.101.132             : ok=6    changed=2    unreachable=0    failed=0   
10.142.97.124              : ok=6    changed=2    unreachable=0    failed=0   
```

## 思考
使用Ansable极大的简化了工作量，而且使得生产的组件管理变得统一，当然这次Ansable的使用很简单，以后有需求可以做很多事情。

## 附 安装脚本

``` sh spark_client_deploy.yml
---
# 自动在多台机器上安装spark客户端
- name: deploy spark client(for version 1.6.2 or 2.2.0)
  vars:
    ansible_ssh_user: op
    ansible_ssh_pass: ******
    spark_auto_path:  /home/op/spark
      spark_version: spark2.2.1
    spark_tar_file: ctcc-spark-2.2.1.tar.gz
    jdk_tar_file: jdk-8u111-linux-x64.tar.gz
  hosts: yaxin3_2.2.1
  gather_facts: false
  sudo: True

  tasks:
    - name: ping
      ping:

    - name: mkdir
      file: path={{spark_auto_path}} state=directory

    - name: test java8 is exist
      shell: '/usr/local/jdk1.8.0_111/bin/java -version'
      register: test_java_result
      ignore_errors: True

    - name: copy java8
      copy: backup=no src=files/{{jdk_tar_file}} force=no dest={{spark_auto_path}}/{{jdk_tar_file }}
      when: test_java_result.rc != 0

    - name: extract java
#       shell: 'sudo tar -xzvf /home/op/spark/jdk-8u111-linux-x64.tar.gz -C /usr/local/' 
      shell: 'sudo tar -xzvf {{spark_auto_path}}/{{jdk_tar_file }} -C /usr/local/'
      when: test_java_result.rc != 0

    - name: copy ctcc-spark
      copy: backup=yes src=files/{{spark_version}}/{{spark_tar_file}} dest={{spark_auto_path}}/{{spark_tar_file}} mode=0755

    - name: rm old spark2
      file: path=/usr/local/spark2.2 state=absent

    - name: extract ctcc-spark
      shell: 'sudo tar -xzvf {{spark_auto_path}}/{{spark_tar_file}} -C /usr/local/'

    # - name: spark home
    #   shell: sudo echo 'export SPARK_HOME=/usr/local/spark2.2'  >> /etc/profile
```