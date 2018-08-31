---
title: 使用scaladiagrams工具构建scala项目的UML图
date: 2016-06-13 21:12:22
tags: scala
categories: scala
---

### 背景

阅读spark源码到storage这一块的时候，由于类的继承，调用之间的关系比较复杂，想要画一下UML图，idea自带的diagrams方法对java支持很好，但对scala的一些继承关系支持不佳，因此google了一下有没有可以画scala UML类图的工具，还真找到了：

我是在x64 windows10下面，使用gitbash工具作为shell命令行，亲测可用

### clone开源项目scaladiagrams并安装
``` bash
git clone https://github.com/mikeyhu/scaladiagrams.git
cd scaladiagrams
./build
```
### 安装graphviz工具
graphviz是一个开源的图形可视化软件，矢量图生成工具，与其他图形软件所不同，它的理念是“所想即所得”，通过dot语言来描述并绘制图形。
http://www.graphviz.org/Download_windows.php 
如上链接下载，然后安装即可，将安装路径加入path中，该工具的目的是通过scaladiagrams工具生成的依赖关系画图；

### 使用

#### 生成依赖关系文件dotFile
``` bash
./scaladiagrams --source "D:\spark-1.6.0\core\src\main\scala\org\apache\spark\storage" > dotFile
```
dotFile文件就是依赖关系的文件：
官方命名为 dot语言，是一个表示图的语言，挺好玩的：

``` scala dot语言
digraph diagram {
"BlockException" [style=filled, fillcolor=burlywood]
  "BlockException" -> "Exception";

"BlockFetchException" [style=filled, fillcolor=burlywood]
  "BlockFetchException" -> "SparkException";

"BlockId" [style=filled, fillcolor=darkorange]
  

"RDDBlockId" [style=filled, fillcolor=burlywood]
  "RDDBlockId" -> "BlockId";

  。。。
```

#### 使用graphviz工具画图

生成svg文件，文件比较大的话建议用这个
``` bash
cat dotFile | dot -Tsvg > spark_storage.svg
```
生成png文件
``` bash
cat dotFile | dot -Tpng > spark_storage.png
```
画的效果部分截图如下（就是图有点扁平）：
![spark_storage_part.png](spark_storage_part.png)

因为好玩，又画了一个spark_core的类图，太大了，不好看，为了部分解决这个问题，只要在dot文件的第一行加入
``` bash
rankdir=RL; 
```
即可使得图片稍微好看一点
![spark_storage_part2.png](spark_storage_part2.png)


引用：
http://stackoverflow.com/questions/7227952/generating-uml-diagram-from-scala-sources
https://github.com/mikeyhu/scaladiagrams
http://www.graphviz.org/About.php
http://www.graphviz.org/pdf/dotguide.pdf
http://www.tonyballantyne.com/graphs.html