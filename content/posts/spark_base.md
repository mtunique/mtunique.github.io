---
title: 【Spark笔记】基本概念
date: 2014-11-28T13:30:27+08:00
category:
- big-data
- spark
tags:
- big-data
- spark
---

## 代码量

Spark: 20,000 LOC

Hadoop 1.0: 90,000 LOC

Hadoop 2.0: 220,000 LOC

![pic](https://dn-mtunique.qbox.me/sparkCodeSize.png)

## 基本概念

`RDD` - resillient distributed dataset 弹性分布式数据集

`Operation` - 作用于RDD的各种操作分为transformation和action

`Job` - 作业，一个JOB包含多个RDD及作用于相应RDD上的各种operation

`Stage` - 一个作业分为多个阶段

`Partition` - 数据分区， 一个RDD中的数据可以分成多个不同的区

`DAG` - Directed Acycle graph, 有向无环图，反应RDD之间的依赖关系

`Narrow dependency` - 窄依赖，子RDD依赖于父RDD中固定的data partition

`Wide Dependency` - 宽依赖，子RDD对父RDD中的所有data partition都有依赖

`Caching Managenment` - 缓存管理，对RDD的中间计算结果进行缓存管理以加快整体的处理速度
<!-- more -->

这些基本概念会反复提到

## 编程模型

Spark应用程序可分两部分：driver部分和executor部分初始化SparkContext和主体程序

### A：driver部分

driver部分主要是对SparkContext进行配置、初始化以及关闭。初始化SparkContext是为了构建Spark应用程序的运行环境，在初始化SparkContext，要先导入一些Spark的类和隐式转换；在executor部分运行完毕后，需要将SparkContext关闭。

### B：executor部分

Spark应用程序的executor部分是对数据的处理，数据分三种：

#### 原生数据
{% blockquote %}
包含输入的数据和输出的数据。
{% endblockquote %}

##### 输入原生数据

1.  scala集合数据集，如Array(1,2,3,4,5)，Spark使用parallelize方法转换成RDD。
2.  hadoop数据集，Spark支持存储在hadoop上的文件和hadoop支持的其他文件系统，如本地文件、HBase、SequenceFile和Hadoop的输入格式。例如Spark使用txtFile方法可以将本地文件或HDFS文件转换成RDD.

##### 输出数据

1.  生成Scala标量数据，如count（返回RDD中元素的个数）、reduce、fold/aggregate；返回几个标量，如take（返回前几个元素）。
2.  生成Scala集合数据集，如collect（把RDD中的所有元素倒入 Scala集合类型）、lookup（查找对应key的所有值）。
3.  生成hadoop数据集，如saveAsTextFile、saveAsSequenceFile

#### RDD(Resilient Distributed Datasets)
{% blockquote %}
是一个容错的、并行的数据结构，可以让用户显式地将数据存储到磁盘和内存中，并能控制数据的分区。同时，RDD还提供了一组丰富的操作来操作这些数据。在这些操作中，诸如map、flatMap、filter等转换操作实现了monad模式，很好地契合了Scala的集合操作。除此之外，RDD还提供了诸如join、groupBy、reduceByKey等更为方便的操作（注意，reduceByKey是action，而非transformation），以支持常见的数据运算。 RDD作为数据结构，本质上是一个只读的分区记录集合。一个RDD可以包含多个分区，每个分区就是一个dataset片段。RDD可以相互依赖。如果RDD的每个分区最多只能被一个Child RDD的一个分区使用，则称之为narrow dependency；若多个Child RDD分区都可以依赖，则称之为wide dependency。不同的操作依据其特性，可能会产生不同的依赖。例如map操作会产生narrow dependency，而join操作则产生wide dependency。
{% endblockquote %}

##### 主要部分组成

1.  `partitions` — partition集合，一个RDD中有多少data partition
2.  `dependencies` — RDD依赖关系
3.  `compute(parition)`— 对于给定的数据集，需要作哪些计算
4.  `preferredLocations` — 对于data partition的位置偏好
5.  `partitioner` — 对于计算出来的数据结果如何分发
