---
title: 如何选择三种Spark API
date: 2016-11-11T14:30:27+08:00
categories:
- big-data
- spark
tags:
- big-data
- spark
---

> 在[这里](http://www.agildata.com/apache-spark-2-0-api-improvements-rdd-dataframe-dataset-sql/)查看 Apache Spark 2.0 Api 的改进：RDD，DataFrame，DataSet和SQL


Apache Spark 正在快速发展，包括更改和添加核心API。最具破坏性的改变之一是dataset。 Spark 1.0 使用RDD API，但是在最近的十二个月引入了两个新不兼容的API。Spark 1.3 引入了完全不同的DataFrame API 而且最近发布的 Spark 1.6 引入了 Dataset API 浏览版。


很多现有的 Spark 开发者想知道是否应该从 RDDs 直接切换到 Dataset API，或者先切换到 DataFrame API。Spark 新手应该选择哪个 API 开始学习。


这篇文章将提供每个API的概述，并且总结了每个的优缺点。 [配套的github repo](https://github.com/AgilData/spark-rdd-dataframe-dataset)提供了例子，可以从这开始实验文中提到的方法。yd


RDD (弹性分布式数据集) API 从 1.0 开始一直存在 Spark 中。 这个接口和 Java 版本 JavaRDD 对于已经完成标准Spark教程的任何开发人员来说都是熟悉的。从开发人员的角度来看，RDD只是一组表示数据的Java或Scala对象。


RDD API 提供很多转换方法来在数据上执行计算，比如 `map()` ， `filter()` ，和 `reduce()` 。这些方法的结果表示转换后的新RDD。 然而，这些方法只是定义要执行的操作，直到调用action方法才执行转换。action 方法比如是： `collect()` 和 `saveAsObjectFile()`.

<!-- more -->

### RDD 转换和action例子

#### Scala:
```scala
rdd.filter(_.age > 21)              // transformation
   .map(_.last)                     // transformation
   .saveAsObjectFile("under21.bin") // action
```

#### java:
```java
rdd.filter(p -> p.getAge() < 21)     // transformation
   .map(p -> p.getLast())            // transformation
   .saveAsObjectFile("under21.bin"); // action
```


RDD 主要的优势是简单且易于理解，因为他们涉及具体的类，提供了熟悉的面向对象的编程风格，而且编译时类型检查。比如，给定包含Person实例的RDD，我们可以通过引用每个Person对象的age属性来按年龄过滤：

### 用RDD按属性过滤的例子
#### Scala:
```scala
rdd.filter(_.age > 21)
```

#### Java:
```java
rdd.filter(person -> person.getAge() > 21)
```


RDD主要的劣势是表现不是特别好。每当 Spark 需要在集群中奋发数据或者将数据写入硬盘，默认试用 Java 序列化（尽管在大多数情况下可以试用 Kryo 作为更快的替代方案）。序列化单个 Java 和 Scala 对象的开销是昂贵的而且需要在节点之间传输数据（每个序列化对象包括类的结构和数据）。还有由创建和销毁单个对象导致的垃圾回收的开销。

## DataFrame API

Spark 1.3引入了一个新的DataFrame API作为Project Tungsten计划的一部分，旨在提高Spark的性能和可扩展性。 DataFrame API引入了一个模式的概念来描述数据，允许Spark管理模式，并且只在节点之间传递数据，而不是使用Java序列化。在单个进程中执行计算时也有优势，因为Spark可以将数据以二进制格式序列化到堆外存储中，然后直接对此堆外存储器执行许多转换，从而避免为数据集中的每一行构建单个对象造成的垃圾回收成本。 因为Spark理解了模式，所以不需要使用Java序列化来编码数据。


DataFrame API与RDD API截然不同，因为它是一个用于构建关系型查询计划的API，Spark的Catalyst优化器随后可以执行它。 对于熟悉构建查询计划的开发人员来说，这种API非常自然，但对于大多数开发人员来说并不自然。 查询计划可以从字符串中的SQL表达式构建或更函数式的方式构建（通过流式API）。

### 用DataFrame按属性过滤的例子
*请注意，这些示例在Java和Scala中具有相同的语法*

#### SQL 风格
```java
df.filter("age > 21");
```

#### 表达式构建器风格
```java
df.filter(df.col("age").gt(21));
```

因为代码按名称引用数据属性，所以编译器不可能捕获任何错误。 如果属性名不正确，那么只有在创建查询计划时才会在运行时检测到错误。


DataFrame API的另一个缺点是它非常以Scala为中心，虽然它支持Java，但支持有限。 例如，从Java对象的现有RDD创建DataFrame时，Spark的Catalyst优化程序无法推断该模式，并假定DataFrame中的任何对象都实现了scala.Product接口。 Scala case类实现了这个接口，因为它们实现了这个接口。

## Dataset API

在Spark 1.6中作为API预览发布的Dataset API旨在提供两个方式中最好的部分; 熟悉的面向对象编程风格和RDD API的编译时类型安全性，但具有Catalyst查询优化器的性能优势。 Dataset 也使用与 DataFrame API 相同的高效堆外存储机制。


当涉及序列化数据时，Dataset API具有在JVM表示（对象）和Spark的内部二进制格式之间转换的编码器的概念。 Spark具有内置的编码器，这些编码器非常先进，它们生成字节码以与堆外数据进行交互，并提供对各个属性的按需访问，而不必对整个对象进行反序列化。 Spark还没有提供用于实现自定义编码器的API，但是计划在将来的版本中提供。


此外，Dataset API设计为与Java和Scala同样工作良好。 当使用Java对象时，重要的是它们完全符合bean。在编写本文附带的示例时，当试图从不完全与bean兼容的Java对象列表中创建Java中的Dataset时，我们遇到了错误。


### 从对象列表创建Dataset

#### scala
```scala
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)
import sqlContext.implicits._
val sampleData: Seq[ScalaPerson] = ScalaData.sampleData()
val dataset = sqlContext.createDataset(sampleData)
```

#### java
```java
JavaSparkContext sc = new JavaSparkContext(sparkConf);
SQLContext sqlContext = new SQLContext(sc);
List data = JavaData.sampleData();
Dataset dataset = sqlContext.createDataset(data, Encoders.bean(JavaPerson.class));
```


使用Dataset API的转换看起来非常像RDD API，并且处理Person类，而不是行的抽象。

### 用Dataset按属性过滤的例子
#### Scala:
```scala
rdd.filter(_.age < 21)
```

#### Java:
```java
rdd.filter(person -> person.getAge() < 21)
```


尽管与RDD代码相似，但是该代码正在构建查询计划，而不是处理单个对象，如果age是唯一访问的属性，那么对象的其余数据将不会从堆外存储中读取。

## 结论
如果你是主要在Java开发，那么在采用DataFrame或Dataset API之前，值得考虑移动到Scala。 虽然有努力支持Java，但是Spark是用Scala编写的，使得处理Java对象变得困难（但不是不可能）。


如果你是在用Scala开发，需要你的代码使用Spark 1.6.0生产，那么DataFrame API显然是最稳定的选择，目前提供最好的性能。


但是，Dataset API预览版看起来很有前景，并提供了一种更自然的代码编写方式。 鉴于Spark的快速发展，很可能这个API将在2016年快速成熟，并成为开发新应用的实际API。

